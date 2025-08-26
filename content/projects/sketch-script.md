+++
title = 'SketchScript'
date = '2025-06-10'
draft = false
tech = 'Haskell, AWS, Terraform'
github = 'https://github.com/HasithDeAlwis/SketchScript'
description = 'Open-source tool to help designers prototype using custom DSL'
demo = 'https://sketchscript.dev/'
hasDetails = true
+++

# What 

SketchScript is a Domain-Specific Language (DSL) built in Haskell that serves as a code-driven alternative to visual mockup tools like Balsamiq or Figma wireframes. It's designed for designers and developers who prefer to create UI mockups programmatically.

# Why
With advancements in LLMs the jobs of designers and front-end developers moves closer and closer.

With tools such as [Figma MCP](https://www.figma.com/blog/introducing-figmas-dev-mode-mcp-server/) coming around the block to be publicly available, designers must learn the basics of CSS layout tools (such as Flex, Grid, etc.), otherwise, the MCP server will be much less effective. 

This tool doubles down as something that designers can use to prototype with (Design as Code) and a tool they can use to learn basic CSS layouts.

# Technical Overview

## Haskell at the Core: Why and How

SketchScript is a monorepo is built around Haskell, chosen for its strong type system, safety, and composability—perfect for building reliable APIs and custom languages. Haskell’s purity and expressive power make it ideal for both backend logic and DSL (Domain-Specific Language) implementation.

### API Architecture: Servant, Types, and Modularity

The API is constructed using the Servant library, which lets you define type-safe web APIs at the type level. Here’s how it all fits together:

- **Type-Level API Definitions:** Each API module (e.g., `User.Api`, `Project.Server`, `File.Server`, `S3.Server`, `Auth.Server`) defines its endpoints as Haskell types. For example, `User.Api` describes endpoints like `"users" :> Get '[JSON] [User]`.
- **Handlers and Servers:** Each endpoint has a corresponding handler (e.g., `User.Handler`, `Project.Handler`) that implements the business logic, often interacting with a PostgreSQL database via the `postgresql-simple` library and connection pools.
- **Authentication:** Auth is handled using `Servant.Auth.Server`, supporting JWTs and cookies, with OAuth flows for third-party login (see `Auth.Handler` and `Auth.Server`).
- **Composition:** The main server (`App.Server`) composes all sub-APIs into a single application, wiring up CORS, environment config, and database pools.
- **Example:** The `Project.Server` module exposes endpoints for listing and updating projects, delegating to handler functions that validate input, query the DB, and return typed responses.

This approach means the entire API surface is checked at compile time, reducing runtime errors and making refactoring safe and easy.

### The Custom DSL: Parsing, Lexing, and Why Haskell Shines

A standout feature is the custom DSL (Domain-Specific Language) for describing mockups or scripts, implemented in the `sketchscript-core` package.

- **Parser Combinators:** The DSL is parsed using the Megaparsec library, a powerful parser combinator framework. Modules like `Parser.Common` define reusable parsers for things like sizing, content, and quoted strings.
- **Lexing and Parsing:** Lexing (tokenizing input) and parsing (building ASTs) are handled together in Megaparsec, with functions like `parseSizing` and `parseQuotationMarks` that combine primitive parsers into higher-level constructs.
- **Types:** The DSL is mapped to strongly-typed Haskell data structures, ensuring that only valid scripts are accepted and making it easy to extend the language.
- **Why Haskell:** Haskell’s type system and purity make it easy to write correct, composable parsers. Megaparsec’s error messages and combinator style are a perfect fit for evolving a DSL.

### Tools Used

- **Haskell:** For all backend, API, and DSL logic.
- **Servant:** Type-safe API definition and server.
- **Megaparsec:** Parser combinators for the DSL.
- **postgresql-simple:** Database access.
- **Amazonka:** AWS integration (S3, etc.).
- **Nx:** Monorepo management.
- **Node.js, Vite, React, Tailwind:** For frontend apps in apps and `website/`.
- **Terraform:** Infrastructure as code.
- **Nix Flakes:** Reproducible dev environments.

## Dev Environment: Nix Flakes for Reproducibility

The repo uses Nix Flakes (flake.nix) to define a reproducible development environment. This means:

- All dependencies (Haskell toolchain, Node, DB clients, etc.) are specified declaratively.
- Onboarding is as simple as running `nix develop`—no more “works on my machine.”
- CI/CD and local dev use the exact same tool versions, reducing surprises.

---

**In summary:**  
Haskell’s type safety and composability make it the perfect choice for building robust APIs and custom DSLs. Servant and Megaparsec provide a type-driven, maintainable foundation for both web and language features, while Nix Flakes ensures every contributor has a consistent, reliable dev environment.



