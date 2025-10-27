# Working with Redpanda Connect YAML Configuration Files with AI

## Setup

### Local Redpanda Connect Code and Docs repositories

You have to have the following repositories checked out to the `repos` directory in the `redpanda-connect-skill`.
- benthos
- connect
- rp-connect-docs

If not present check out repositories with:
```bash
mkdir -p repos
cd repos
git clone --single-branch --depth 1 git@github.com/redpanda-data/benthos.git
git clone --single-branch --depth 1 git@github.com/redpanda-data/connect.git
git clone --single-branch --depth 1 git@github.com/redpanda-data/rp-connect-docs.git
```

### RPK command line program

The `rpk` command line must be installed. If not present the installation instructions can be found in https://docs.redpanda.com/current/get-started/rpk-install/.
