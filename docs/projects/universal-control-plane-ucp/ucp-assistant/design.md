---
title: "Design"
space: UCP
parent_page_id: "../ucp-assistant.md"
---

# UCP Assistant Design

## Overview

UCP Assistant is a session-based, multi-source documentation assistant for the
Universal Control Plane (UCP) domain. It retrieves trusted documentation at
answer time, generates grounded answers from that evidence, and appends
references to the documents used.

The assistant supports multiple clients through one shared backend. The CLI is
the primary interface. Slack, Teams, web, and API clients use the same core
assistant engine and the same answer contract.

---

## Design Principles

### Grounded by Default

The assistant answers from retrieved documentation, not from unaudited model
memory. Retrieval is mandatory for factual answers.

### Multi-Source by Design

The assistant indexes multiple repositories and document collections. Each source
retains its own identity, metadata, and provenance.

### Session-Based Conversations

The assistant stores conversation state per session so follow-up questions work
consistently across CLI, API, and messaging interfaces.

### Consistent Response Contract

The assistant uses one answer structure across all interfaces and always appends
references at the end.

### Provider-Agnostic Runtime

The assistant uses a pluggable LLM provider layer. The local CLI runs against
whichever provider a developer configures on their machine, as long as the
provider exposes a supported API.

### Source Traceability

Every indexed chunk preserves source metadata so answers can point back to a
specific repository, document path, heading, and commit.

---

## Knowledge Model

The assistant treats each knowledge source as a document collection with explicit
metadata.

Recommended metadata per indexed chunk:

- `source_name`
- `source_type`
- `repo_url`
- `branch`
- `commit_sha`
- `document_path`
- `document_title`
- `section_heading`
- `doc_type`
- `product`
- `last_synced_at`

This metadata supports:

- source filtering
- source comparison
- conflict reporting
- reference generation
- commit-aware traceability

---

## Core RAG Model

Retrieval-Augmented Generation (RAG) is the answer path for the assistant.

Core flow:

1. ingest trusted source documents
2. normalize document content
3. split content into chunks
4. index chunks for lexical and semantic retrieval
5. retrieve relevant chunks for each question
6. pass retrieved context to the model
7. generate an answer from retrieved context only
8. append references to the source documents

The assistant reads documentation at answer time. It does not rely on the model
to remember internal documentation without retrieval.

---

## Grounding Rules

The grounding policy defines how the assistant answers and when it refuses to
answer beyond the evidence.

### Required Behavior

- retrieval runs before every answer
- the answer is based only on retrieved documentation
- unsupported claims are not inferred
- missing information is stated explicitly
- references appear at the end of every answer

### Supported Answer Cases

If retrieved documentation clearly supports the answer:

- provide a direct answer
- include relevant caveats
- append references

If retrieved documentation partially supports the answer:

- answer only the supported portion
- state what is not documented or not clear
- append references

If retrieved documentation does not support the answer:

- state that the answer cannot be verified from indexed documentation
- do not guess
- append any relevant references only if they help explain the limitation

### Retrieval Guardrails

- hybrid retrieval combines lexical and semantic search
- reranking orders candidate chunks by relevance
- evidence thresholds prevent weak retrieval from producing confident answers
- chunk metadata remains visible to the generation layer
- citations remain mandatory

---

## Answer Contract

The assistant uses a fixed response structure for consistency and trust.

Required sections:

1. `Answer`
2. `Notes`
3. `References`

### Format

`Answer`

- direct response to the question
- concise and technical
- limited to what the retrieved docs support

`Notes`

- caveats
- missing details
- scope limits
- conflicts across sources when present

`References`

- source name
- document title or heading
- document path
- commit SHA or version identifier
- optional direct repository URL

### Example

```text
Answer
UCP uses X for Y. The documented steps are ...

Notes
The indexed documentation does not specify whether this applies to legacy
clusters.

References
1. UCP Docs > Networking > Load Balancer Setup
   source: ucp-docs
   path: docs/networking/load-balancer.md
   commit: abc123def4
```

The `References` section is mandatory. The application layer enforces its
presence so the contract does not depend only on prompt behavior.

---

## Session Model

The assistant is session-based across all interfaces.

Each session stores:

- session ID
- user identity where available
- conversation history
- recent retrieval context
- timestamps
- active source filters

Session context supports follow-up questions, but session memory does not replace
retrieval. Documentation retrieval remains the source of truth on every turn.

---

## Source Synchronization

The assistant synchronizes documentation from read-only sources.

For Git-based sources, synchronization uses:

- `git clone` on first sync
- `git fetch` on subsequent syncs
- branch tracking per configured source
- re-indexing after source changes

The synchronization model is compatible with environments where the assistant has
read access to repositories but no write access.

Synchronization modes:

- manual sync for local usage
- scheduled polling for shared deployments
- periodic full refresh for index consistency

---

## Ingestion Model

Each source definition includes configuration for discovery, parsing, and
reference generation.

Recommended source fields:

- source name
- source type
- clone URL
- branch
- authentication method
- include globs
- exclude globs
- reference URL base
- sync interval
- source tags

Supported initial source type:

- read-only Git repository

Additional source types:

- static website mirror
- wiki export
- PDF collection

---

## Retrieval Pipeline

The retrieval pipeline transforms source documents into grounded answer context.

### Ingestion Steps

1. fetch source content
2. discover eligible files
3. parse Markdown and related text formats
4. extract titles, headings, and metadata
5. split content into chunks
6. build lexical and semantic indexes
7. persist chunk provenance

### Query Steps

1. receive user question
2. load session context and source filters
3. retrieve candidate chunks
4. rerank candidate chunks
5. send selected evidence to the model
6. generate answer in the required format
7. validate that references are present
8. return the answer to the client

---

## Interface Model

The assistant engine is separate from interface clients.

### CLI

The CLI provides the primary user experience.

Recommended commands:

- `ucp-assistant chat`
- `ucp-assistant ask "<question>"`
- `ucp-assistant search "<query>"`
- `ucp-assistant sync`
- `ucp-assistant status`

Recommended chat commands:

- `/help`
- `/sources`
- `/search`
- `/sync`
- `/status`
- `/clear`
- `/source <name>`

### API

The API exposes the same assistant capabilities to other clients.

Recommended API operations:

- create session
- send message
- search indexed documents
- synchronize sources
- fetch assistant status

### Messaging Clients

Slack, Teams, and similar interfaces act as thin clients over the same
session-aware backend. Retrieval, answer formatting, and source handling remain
centralized in the assistant engine.

---

## Runtime and Installation Model

The assistant runs as a local CLI application and as a shared backend service.

### CLI Installation

The CLI is distributed as an installable command-line application.

Primary installation path:

- Homebrew tap hosted in GitHub Enterprise

Additional installation patterns:

- standalone packaged binary
- Python package installation into a virtual environment for development use

The installed CLI exposes commands such as:

- `ucp-assistant chat`
- `ucp-assistant ask`
- `ucp-assistant search`
- `ucp-assistant sync`
- `ucp-assistant status`

### Homebrew Distribution Model

The CLI is distributed through an internal Homebrew tap so developers install
and upgrade it through standard Brew workflows.

Distribution components:

- main CLI source repository in GitHub Enterprise
- release artifacts for supported platforms
- Homebrew tap repository containing the formula

The Homebrew installation flow is:

1. tap the internal repository
2. install `ucp-assistant`
3. configure provider and source credentials locally
4. run the CLI

The update flow uses:

- `brew update`
- `brew upgrade ucp-assistant`

### Local CLI Configuration

Each developer configures the CLI locally with:

- LLM provider
- provider base URL where applicable
- chat model
- embedding model when semantic retrieval uses a separate model
- provider API key
- source access credentials

The CLI stores local runtime configuration outside indexed source content.

Recommended configuration fields:

- `provider`
- `base_url`
- `api_key`
- `model`
- `embedding_provider`
- `embedding_base_url`
- `embedding_api_key`
- `embedding_model`

Supported provider forms:

- company AI gateway
- OpenAI-compatible API
- direct provider integration
- future custom provider adapters

The installation mechanism and the provider configuration are separate concerns.
Homebrew installs the CLI binary. Local runtime configuration selects the LLM
provider, model, and credentials used by that binary.

### Backend Runtime

The shared deployment runs as a long-lived service with:

- source synchronization
- retrieval index storage
- session storage
- AI provider integration
- HTTP API for client access

---

## Credential Model

The assistant requires credentials for two separate concerns:

1. source access credentials
2. LLM provider credentials

### Source Access Credentials

Source access credentials allow the assistant to clone and fetch read-only
documentation repositories.

Supported forms:

- GitHub Enterprise read-only credentials
- Bitbucket read-only credentials
- SSH deploy keys
- HTTPS personal access tokens with read scope

### LLM Provider Credentials

The chat and answer-generation features require credentials for the configured
LLM provider. Semantic retrieval also requires provider credentials when
embeddings are generated through an external API.

Recommended configuration methods:

- environment variable
- secret manager
- deployment platform secret store

Required local provider settings:

- provider identifier
- API key or access token
- base URL when required by the provider
- model identifier

The credential contract is provider-agnostic. A company AI gateway, a direct
provider endpoint, or another approved internal endpoint can satisfy this
contract as long as it exposes a supported API shape.

### Client Authentication

The CLI uses locally configured credentials. Shared API deployments use service
authentication for inbound client requests and separate server-side LLM provider
credentials for outbound model requests.

This keeps end-user clients from directly handling provider secrets in shared
deployment modes.

---

## Trust Model

The assistant earns trust through explicit evidence and predictable behavior.

Trust signals:

- references at the end of every answer
- plain statements when documentation is missing or unclear
- source identity in every citation
- document path and heading in references
- commit SHA or version context where available

The assistant prefers a constrained response over a plausible unsupported answer.

---

## Architecture

The assistant consists of these core components:

1. source sync manager
2. document parser and chunker
3. retrieval index
4. answer generation layer
5. session manager
6. CLI client
7. HTTP API
8. messaging clients

Separation of concerns:

- ingestion is separate from retrieval
- retrieval is separate from generation
- core assistant logic is separate from interface clients
- session state is managed centrally

This structure keeps behavior consistent across terminal and messaging
environments.
