# Architecture Blueprint for Payload Consequence Analyzer

## New Directory Structure

```plaintext
payload-consequence-analyser/
├── analyzers/
│   ├── golang/
│   │   ├── example.go
│   ├── typescript/
│   │   ├── example.ts
│   ├── registry/
│   └── common/
├── cli/
│   ├── cli.go
│   └── cli.ts
├── docs/
└── tests/
```  

## Code Examples for Multi-Language Support

### Go Example

```go
package main

import "fmt"

func Main() {
    fmt.Println("Welcome to Go Analyzer")
}

func ExampleAnalysis(input string) string {
    // Implementation goes here
    return ""  
}
```

### TypeScript Example

```typescript
function main(): void {
    console.log('Welcome to TypeScript Analyzer');
}

function exampleAnalysis(input: string): string {
    // Implementation goes here
    return '';
}
```  

## Analyzer Registry Pattern

The analyzer registry will manage the registration of different analyzers across languages, allowing for expansion and easier management. Each analyzer can implement a common interface to standardize behavior.

### Registry Interface

```go
package registry

type Analyzer interface {
    Analyze(input string) string
}

type Registry struct {
    analyzers map[string]Analyzer
}

func (r *Registry) Register(name string, analyzer Analyzer) {
    r.analyzers[name] = analyzer
}
}
```

### CLI Refactoring

The CLI will allow users to choose between analyzers dynamically at runtime, providing an additional flag for the language type.

### CLI Example in Go

```go
package main

import "flag"

func main() {
    var lang string
    flag.StringVar(&lang, "lang", "go", "Choose the language analyzer")
    flag.Parse()

    // Logic to select and run the appropriate analyzer based on the 'lang' flag
}
```  

### CLI Example in TypeScript

```typescript
import * as yargs from 'yargs';

yargs.command({
    command: 'analyze',
    describe: 'Run analyzer',
    builder: {
        lang: {
            describe: 'Choose the language analyzer',
            demandOption: true,
            type: 'string',
        },
    },
    handler(argv) {
        // Handler logic here
    },
});
```  

## Implementation Steps for Go and TypeScript Analyzers

1. **Define Interfaces**: Create common interfaces for analyzers in both languages.
2. **Implement Analyzers**: Implement the required analyzers in each language following the base interfaces.
3. **Setup CLI**: Refactor the existing command-line interface to support multiple analyzers.
4. **Testing**: Write and execute tests for each implementation to ensure compatibility and correctness.
5. **Documentation**: Update documentation to reflect changes and provide usage examples.

## Conclusion
This blueprint outlines the roadmap for a structured refactoring of the payload consequence analyzer project, supporting multiple languages, and streamlining the development process through a common registry pattern and a robust CLI.