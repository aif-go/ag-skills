---
name: ag-skills
description: This skill should be used when the user asks to create an ag-core microservice, define a protobuf API, generate code with aggo, implement service logic, add Kitex/Hertz adapter, build a gRPC/HTTP service with ag-core, use aggo new/proto commands, or work with ag-core .proto files, internal/service/, idl/api/ directories.
version: 1.0.0
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
---

# ag-core Skills for AI Agents

This skill provides comprehensive ag-core microservices framework knowledge, optimized for AI agents helping developers build production-ready services with protobuf IDL, Kitex (gRPC), and Hertz (HTTP).

## When to Use This Skill

Invoke when working with ag-core:
- **Initializing projects**: `aggo new` + AI context injection (see [create-project](workflows/create-project.md))
- **Defining APIs**: protobuf IDL with gRPC + HTTP annotations
- **Generating code**: `aggo proto` with all plugin/mode combinations
- **Implementing business logic**: `internal/service/` layer
- **Calling other services**: generating and using client code

## Knowledge Structure

Load specific guides as needed:

### Workflow Guides

#### Create Project
**File**: [workflows/create-project.md](workflows/create-project.md)
**When**: User asks to create/init/scaffold a new ag-core project

### Pattern Guides

#### 1. Proto IDL Patterns
**File**: [references/proto-idl-patterns.md](references/proto-idl-patterns.md)
**When**: Writing .proto files, defining gRPC services, adding HTTP annotations
**Contains**: Proto syntax, service patterns, HTTP method annotations, path params

#### 2. Code Generation
**File**: [references/code-generation.md](references/code-generation.md)
**When**: Running `aggo proto`, understanding generation targets, post-generation steps
**Contains**: Full command reference, plugin/mode matrix, output file mapping

#### 3. Project Structure
**File**: [references/project-structure.md](references/project-structure.md)
**When**: Understanding project layout, knowing where to write code vs generated files
**Contains**: Directory tree, file responsibilities, modification rules

#### 4. Kitex Patterns
**File**: [references/kitex-patterns.md](references/kitex-patterns.md)
**When**: Using Kitex for gRPC communication, adding interceptors, configuring gRPC server/client
**Contains**: gRPC server/client architecture, middleware priority system, metadata propagation, fx injection

#### 5. Hertz Patterns
**File**: [references/hertz-patterns.md](references/hertz-patterns.md)
**When**: Using Hertz for HTTP services, adding middleware, routing, request validation
**Contains**: HTTP routing, ServerConfigurator assembly, middleware injection, request binding, client calls

#### 6. Database Patterns
**File**: [references/database-patterns.md](references/database-patterns.md)
**When**: Database access, Repository layer, transactions, GORM integration
**Contains**: Repository.DB(ctx) pattern, transaction propagation, AOP transactions, pagination, fx injection

### Supporting Resources

#### Best Practices
**File**: [best-practices/overview.md](best-practices/overview.md)
**When**: Production deployment, code quality, security hardening

#### Troubleshooting
**File**: [troubleshooting/common-issues.md](troubleshooting/common-issues.md)
**When**: aggo proto errors, compile failures, runtime issues, debugging

## Core Workflows

### Initializing a New Project (with AI Context)

1. Follow [workflows/create-project.md](workflows/create-project.md)
2. This includes: `aggo new` → inject `.claude/` AI context → verify build

### Creating a New Service (in Existing Project)

1. Define proto in `idl/api/<service>/`
2. Generate code with `aggo proto` (one command, see code-generation.md)
3. Implement logic in `internal/service/agservice_*.go`
4. `go mod tidy && go build ./...`

### Adding a New API Service

1. Create `idl/api/<service>/<service>.proto`
2. Define gRPC service + HTTP annotations
3. Run full `aggo proto` pipeline
4. Implement in `internal/service/`

### Calling Other Microservices

1. Copy callee's `.proto` to `idl/api/<service>/`
2. `aggo proto -p hertz -m client ...` and `-p kitex -m client ...`
3. Use generated client code in service layer

## Key Principles

### Always Follow
- Proto-First: define `.proto` before any code
- Service layer only: business logic in `internal/service/`
- Never edit generated code (adpgen/ svcgen/)
- Post-generation: `go mod tidy && go build ./...`

### Never Do
- Put business logic outside `internal/service/`
- Manually edit `adpgen/` or `svcgen/` files
- Skip `go mod tidy` after generation
- Forget HTTP annotations when generating Hertz code
