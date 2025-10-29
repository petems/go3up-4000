# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Go3Up (Go S3 Uploader) is a command-line tool that efficiently uploads files to S3 by maintaining a local cache of MD5 checksums. It only uploads files that have changed since the last run, significantly speeding up deployments for large static sites where only a small subset of files typically changes.

## Common Commands

### Building
```bash
make build
# or
go build
```

### Testing
```bash
# Run all tests with race detection and coverage
make test

# Run tests with coverage report (opens in browser)
make cover

# Run single test
AWS_SECRET_ACCESS_KEY=secret AWS_ACCESS_KEY_ID=secret go test -race -run TestName
```

**Important**: AWS credentials must be set for tests (they can be dummy values like "secret" as shown above).

### Running the Application
```bash
# Run with default test configuration
make run

# Manual run example
./go3up -bucket="bucket-name" -source=output -cachefile=.go3up.txt
```

## Architecture

### Core Components

**Main Upload Flow** (main.go:142-206):
1. Validates command-line flags and loads config from `.go3up.json` if present
2. Initializes AWS S3 client using credential chain: shared profile → EC2 role → environment variables
3. Computes file diff by comparing current file hashes with cached hashes
4. Spawns worker pool (default: NumCPU * 2) to upload files concurrently
5. Implements exponential backoff retry logic (up to 10 attempts) for recoverable S3 errors
6. Updates local cache file with successfully uploaded file hashes

**File Hashing & Caching**:
- Uses MD5 checksums stored in `.go3up.txt` (configurable via `-cachefile`)
- Only files that changed since last run are uploaded
- Cache is updated only after successful uploads to avoid inconsistent state
- Failed uploads are excluded from cache update via rejected list (synced_list.go)

**Concurrent Upload Workers** (main.go:48-84):
- Workers pull from `uploads` channel and process files in parallel
- Retry logic with exponential backoff: 100ms * 2^attempts
- Non-recoverable errors (see utils.go:10-17) immediately fail without retry
- Thread-safe rejected list tracks failed uploads

**Custom HTTP Headers** (setup.go:42-51):
- Pattern-based header mapping using regex matching (first match wins)
- Supports Content-Encoding (gzip), Cache-Control, and S3 server-side encryption
- Files matching gzip patterns are compressed during upload using pipe + gzip writer
- Content-Type automatically detected from file extension

**Configuration** (opts.go):
- CLI flags override config file values
- Config can be saved with `-save` flag to `.go3up.json`
- Defaults: 2*NumCPU workers, "output" source dir, ".go3up.txt" cache file

### Key Design Patterns

**Worker Pool Pattern**: Fixed number of goroutines processing uploads from shared channel, with WaitGroups for synchronization.

**Retry with Backoff**: Failed uploads are re-enqueued with exponential delay, tracked via sourceFile.attempts counter.

**Thread-Safe Rejected List**: syncedlist.go provides mutex-protected list for tracking failed uploads across concurrent workers.

**Gzip On-The-Fly**: For files marked with Content-Encoding:gzip header, compression happens during upload using io.Pipe to stream compressed data without temporary files (main.go:105-122).

## Development Notes

- Target Go version: 1.13+
- AWS SDK errors ending in specific suffixes (see utils.go:10-17) are considered recoverable and will be retried
- Tests use dummy AWS credentials and skip actual S3 operations when appEnv == "test"
- The custom headers mapping in setup.go:42-51 is hardcoded but marked with TODO for making it configurable
