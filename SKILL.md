
# Redpanda Connect YAML Configuration Skill

The purpose of this document is to instruct you on how to work with Redpanda Connect YAML configuration files.
How to write, validate and debug YAML configuration files and bloblang scripts.
Do not write README files for the pipelines, just the pipelines.

## Discover Components

### Categories

Component categories:

- bloblang-functions
- bloblang-methods
- buffers
- caches
- inputs
- metrics
- outputs
- processors
- rate-limits
- scanners
- tracers

The most important categories for YAML configuration are:

- inputs
- processors
- outputs

### List Components

You can list all components in a category with:

```bash
rpk connect list <category>
```

Example:
```bash
rpk connect list inputs
rpk connect list processors
rpk connect list outputs
```

## Get Component JSON Schema

You can get a component JSON schema with:

```bash
rpk connect list --format json <category> <component>
```

Example:
```bash
rpk connect list --format json inputs mongodb
rpk connect list --format json processors http
rpk connect list --format json outputs redpanda
```

Another example:
```bash
rpk connect list --format json inputs sql_select
```

Use JSON schema to understand all available fields and types for YAML config.

## Validate YAML

You can validate YAML configuration with:

```bash
rpk connect lint <path/to/config.yaml>
```

## Local Resources

All code and docs resources should be checked out and available in `repos` directory.

- **Component reference docs**: `repos/rp-connect-docs/modules/components/`
- **Config examples**: `repos/connect/config/examples/`
- **Template examples**: `repos/connect/config/template_examples/`
- **Test configs**: `repos/connect/config/test/`
- **Docs fallback**: `repos/connect/docs/modules/`

## YAML Structure

### Basic Pipeline

```yaml
input:
  <input_type>:
    # input fields from JSON schema

pipeline:
  processors:
    - <processor_type>:
        # processor fields from JSON schema

output:
  <output_type>:
    # output fields from JSON schema
```

### With Resources (Reusable Components)

```yaml
input_resources:
  - label: my_input
    <input_type>:
      # configuration

processor_resources:
  - label: my_processor
    <processor_type>:
      # configuration

output_resources:
  - label: my_output
    <output_type>:
      # configuration

input:
  resource: my_input

pipeline:
  processors:
    - resource: my_processor

output:
  resource: my_output
```

## YAML Examples

### Simple Kafka Consumer

```yaml
input:
  kafka_franz:
    seed_brokers: ["localhost:9092"]
    topics:
      - '^[^_]'  # Skip topics starting with _
    regexp_topics: true
    start_from_oldest: true
    consumer_group: foobar

output:
  stdout: {}
  processors:
    - mapping: |
        root = this.merge({
          "count": counter(),
          "topic": @kafka_topic,
          "partition": @kafka_partition
        })
```

### HTTP Metrics

```yaml
input:
  http_server:
    address: 0.0.0.0:4196
    path: /poke
    allowed_verbs: [POST, HEAD]
    cors:
      enabled: true
      allowed_origins: ['*']
  processors:
    - metric:
        type: counter
        name: site_visit
        labels:
          host: ${! meta("h") }
          path: ${! meta("p") }
          referrer: ${! meta("r") }
    - bloblang: 'root = deleted()'

metrics:
  mapping: |
    root = if !["site_visit"].contains($path) { deleted() } else { $path }
  prometheus: {}
```

### Switch/Case Logic

```yaml
input:
  discord:
    channel_id: ${DISCORD_CHANNEL:xxx}
    bot_token: ${DISCORD_BOT_TOKEN:xxx}
    cache: request_tracking
    cache_key: last_message_received

pipeline:
  processors:
    - switch:
        - check: this.type == 7
          processors:
            - bloblang: |
                root = "Welcome <@%v>!".format(this.author.id)

        - check: this.content == "/joke"
          processors:
            - bloblang: |
                let jokes = [
                  "What do you call a belt made of watches? A waist of time.",
                  "What does a clock do when it's hungry? It goes back four seconds."
                ]
                root = $jokes.index(timestamp_unix_nano() % $jokes.length())

        - check: this.content == "/release"
          processors:
            - bloblang: 'root = ""'
            - try:
              - http:
                  url: https://api.github.com/repos/redpanda-data/benthos/releases/latest
                  verb: GET
              - bloblang: 'root = "Latest release: %v".format(this.tag_name)'

        - processors:
            - bloblang: 'root = deleted()'

    - catch:
      - log:
          fields_mapping: 'root.error = error()'
          message: "Failed to process message"
      - bloblang: 'root = "Error occurred"'

output:
  discord:
    channel_id: ${DISCORD_CHANNEL:xxx}
    bot_token: ${DISCORD_BOT_TOKEN:xxx}

cache_resources:
  - label: request_tracking
    file:
      directory: /tmp/discord_bot
```

### CDC Replication

```yaml
input:
  postgres_cdc:
    dsn: postgres://user:pass@localhost:5432?sslmode=disable
    include_transaction_markers: true
    slot_name: test_slot
    stream_snapshot: true
    schema: public
    tables: [my_src_table]
    batching:
      check: '@operation == "commit"'
      period: 10s
      processors:
        - mapping: |
            root = if @operation == "begin" || @operation == "commit" {
              deleted()
            } else {
              this
            }

output:
  switch:
    strict_mode: true
    cases:
      - check: '@operation != "delete"'
        output:
          sql_raw:
            driver: postgres
            dsn: postgres://user:pass@localhost:5432?sslmode=disable
            args_mapping: root = [this.id, this.foo, this.bar]
            query: |
              MERGE INTO dst_table AS old
              USING (SELECT $1 id, $2 foo, $3 bar) AS new
              ON new.id = old.id
              WHEN MATCHED THEN UPDATE SET foo = new.foo
              WHEN NOT MATCHED THEN INSERT (id, foo, bar) VALUES (new.id, new.foo, new.bar);

      - check: '@operation == "delete"'
        output:
          sql_raw:
            driver: postgres
            dsn: postgres://user:pass@localhost:5432?sslmode=disable
            query: DELETE FROM dst_table WHERE id = $1
            args_mapping: root = [this.id]
```

## YAML Writing Workflow

1. **Find input component**: `rpk connect list inputs`
2. **Get schema**: `rpk connect list --format json inputs <component>`
3. **Find output component**: `rpk connect list outputs`
4. **Get schema**: `rpk connect list --format json outputs <component>`
5. **Write YAML**: Use schema fields to build config
  - Do not use deprecated components
  - Do not use deprecated fields
  - Minimize fields; avoid empty or default fields. Add only required fields
6. **Validate**: `rpk connect lint config.yaml`

## YAML Patterns

**Environment variables**: `${VAR_NAME:default_value}`

**Metadata access**: `@kafka_topic`, `@kafka_partition`, `meta("key")`

**Bloblang interpolation**: `${! bloblang_expression }`

**Switch/case**: Use `switch` processor with `check` conditions

**Error handling**: Use `catch` processor for error recovery

**Batching**: Use `batching` on inputs with `check` condition

**Resources**: Define reusable components with `label` in `*_resources` sections

## Real Examples Location

Browse `repos/connect/config/examples/` for production patterns:
- `discord_bot.yaml` - Switch/case, HTTP calls, error handling
- `cdc_replication.yaml` - CDC with batching and switch output
- `site_analytics.yaml` - Metrics collection
- `stateful_polling.yaml` - Stateful processing
- `joining_streams.yaml` - Stream joins

## Bloblang

Bloblang is the primary data transformation language in Redpanda Connect.

### List Bloblang Functions

```bash
rpk connect list bloblang-functions
```

### List Bloblang Methods

```bash
rpk connect list bloblang-methods
```

### Test Bloblang Scripts

Use `blobl` command to test mappings with sample data.

**IMPORTANT**: The input JSON must end with a newline character, otherwise `blobl` will not produce output.

#### Testing Single-Line Mapping

```bash
# Using printf (recommended - ensures newline)
printf '{"foo":"bar"}\n' | rpk connect blobl 'root.foo = this.foo.uppercase()'

# Using echo (adds newline automatically)
echo '{"foo":"bar"}' | rpk connect blobl 'root.foo = this.foo.uppercase()'

# Using heredoc (adds newline automatically)
rpk connect blobl 'root.foo = this.foo.uppercase()' <<< '{"foo":"bar"}'
```

#### Testing Multi-line Mappings

For multi-line mappings, always use a file:

```bash
# Create mapping file
cat > test.blobl <<'EOF'
let decoded = this.bar.decode("hex")
let value = $decoded.number()
root = this.merge({"result": $value})
EOF

# Test it
printf '{"bar":"64"}\n' | rpk connect blobl -f test.blobl
```

### Bloblang Implementation

Function implementations are in:
- `repos/benthos/internal/impl/io/bloblang.go` - I/O functions (hostname, env, file)
- `repos/benthos/internal/impl/pure/` - Pure functions
- `repos/connect/internal/impl/*/bloblang.go` - Component-specific functions

Functions are registered with `bloblang.MustRegisterFunctionV2`:

```go
bloblang.MustRegisterFunctionV2("hostname",
    bloblang.NewPluginSpec().
        Impure().
        Category(query.FunctionCategoryEnvironment).
        Description(`Returns hostname of the machine.`).
        Example("", `root.host = hostname()`),
    func(_ *bloblang.ParsedParams) (bloblang.Function, error) {
        return func() (any, error) {
            h, err := os.Hostname()
            if err != nil {
                return nil, err
            }
            return h, nil
        }, nil
    },
)
```

Methods are registered with `bloblang.MustRegisterMethodV2`:

```go
bloblang.MustRegisterMethodV2("compress",
    bloblang.NewPluginSpec().
        Impure().
        Category(query.FunctionCategoryEnvironment).
        Description(`Compresses the input document using gzip.`).
        Example("", `root.compressed = compress(this)`),
    func(_ *bloblang.ParsedParams) (bloblang.Method, error) {
        return func() (any, error) {
            return gzip.compress(this)
        }, nil
    },
)
```

### Bloblang Basics

**Assignment**: `root.field = expression`

**This context**: `this` refers to input document

**Metadata**: Access with `@` prefix or `meta()` function

**Variables**: Define with `let`, reference with `$`

```bloblang
let name = this.user.name
root.greeting = "Hello " + $name
```

**Conditionals**: `if/else` expressions

```bloblang
root.status = if this.age >= 18 { "adult" } else { "minor" }
```

**Deletion**: Use `deleted()` to remove fields

```bloblang
root = this
root.sensitive_field = deleted()
```

### Common Bloblang Patterns

**Copy and modify**:
```bloblang
root = this
root.timestamp = now()
root.id = uuid_v4()
```

**Conditional field**:
```bloblang
root = this
root.category = if this.score > 80 { "high" } else { "low" }
```

**Array transformation**:
```bloblang
root.items = this.items.map_each(item -> item.uppercase())
root.filtered = this.items.filter(item -> item.score > 50)
```

**Array folding (reduce)**:
```bloblang
# Sum array values
let sum = this.numbers.fold(0, item -> item.tally + item.value)

# Product of array values
let product = this.numbers.fold(1, item -> item.tally * item.value)

# Convert byte array to integer (big-endian)
let int_value = range(0, this.bytes.length()).fold(0, item -> item.tally * 256 + this.bytes.index(item.value))

# Note: item.tally is the accumulator, item.value is the current element
```

**Metadata access**:
```bloblang
root = this
root.topic = @kafka_topic
root.partition = @kafka_partition
root.custom = meta("custom_key")
```

**Error handling**:
```bloblang
root.value = this.field.catch("default")
root.parsed = this.json_field.parse_json().catch({})
```

## Bloblang Writing Workflow

1. List available functions and methods
2. Write bloblang script
3. Test bloblang script with JSON data using `printf '{"data":"value"}\n' | rpk connect blobl -f script.blobl`
4. Iterate, if more information is needed read the function implementation
