# Blackbox Exporter Explained

This document provides an overview of the technology stack used in the Blackbox Exporter and a class diagram illustrating its main modules and their relationships.

## Technology Stack

The Blackbox Exporter is primarily built with Go and leverages several libraries and tools from the Prometheus ecosystem and the broader Go community.

*   **Language:**
    *   **Go:** The core programming language used for the exporter.

*   **Key Go Libraries & Frameworks:**
    *   **`github.com/prometheus/client_golang`:** The official Go client library for instrumenting applications with Prometheus metrics. Used to create and expose metrics like `probe_success` and `probe_duration_seconds`.
    *   **`github.com/prometheus/common`:** Common libraries for Prometheus components, including logging (`promlog`) and version information.
    *   **`github.com/prometheus/exporter-toolkit`:** A collection of tools and packages to build Prometheus exporters, simplifying tasks like HTTP serving and flag parsing.
    *   **`github.com/go-kit/log`:** A structured logging toolkit.
    *   **`gopkg.in/alecthomas/kingpin.v2`:** A flexible command-line flag parsing library. Used to define and parse flags like `--config.file` and `--web.listen-address`.
    *   **`gopkg.in/yaml.v3` (and `v2`):** Libraries for parsing and marshalling YAML data, used for the `blackbox.yml` configuration file.
    *   **`github.com/miekg/dns`:** A DNS library used for the DNS prober.
    *   **`google.golang.org/grpc`:** The Go implementation of gRPC, used for the gRPC prober.
    *   Standard Go libraries for HTTP, networking, OS interaction, etc.

*   **Configuration:**
    *   **YAML (`blackbox.yml`):** The primary method for configuring probe modules.

*   **Continuous Integration & Delivery (CI/CD):**
    *   **CircleCI:** Used for running tests and builds (see `.circleci/config.yml`).
    *   **GitHub Actions:** Used for tasks like linting (see `.github/workflows/golangci-lint.yml`).

*   **Containerization:**
    *   **Docker:** A `Dockerfile` is provided, allowing the Blackbox Exporter to be built and run as a Docker container.

## Module Relationships (Class Diagram)

The following diagram illustrates the main components of the Blackbox Exporter and how they interact.

```mermaid
classDiagram
    class Main {
        +main()
        +run()
        -configFile: string
        -listenAddress: string
        -sc: SafeConfig
        -rh: ResultHistory
        -setupHTTPServer()
        -handleProbeRequest(http.ResponseWriter, *http.Request)
    }

    class SafeConfig {
        +C: *Config
        +ReloadConfig(string, log.Logger) error
        #Lock()
        #Unlock()
    }

    class Config {
        +Modules: Map<string, Module>
    }

    class Module {
        +Prober: string
        +Timeout: time.Duration
        +HTTP: HTTPProbe
        +TCP: TCPProbe
        +DNS: DNSProbe
        +ICMP: ICMPProbe
        +GRPC: GRPCProbe
        +Headers: Map<string, string>
        # (other prober-specific configs)
    }

    class ProberHandler {
        +Handler(http.ResponseWriter, *http.Request, *Config, log.Logger, *ResultHistory, float64, url.Values)
        -getTimeout(*http.Request, Module, float64) float64
        -selectProber(string) ProbeFn
        -executeProbe(ProbeFn, ...) bool
    }

    class ProbeFn <<(F,orchid) Function Type>> {
        +func(context.Context, string, Module, *prometheus.Registry, log.Logger) bool
    }

    class ProbeHTTP {
        +ProbeHTTP(context.Context, string, Module, *prometheus.Registry, log.Logger) bool
    }

    class ProbeTCP {
        +ProbeTCP(context.Context, string, Module, *prometheus.Registry, log.Logger) bool
    }

    class ProbeDNS {
        +ProbeDNS(context.Context, string, Module, *prometheus.Registry, log.Logger) bool
    }
    
    class ProbeICMP {
        +ProbeICMP(context.Context, string, Module, *prometheus.Registry, log.Logger) bool
    }

    class ProbeGRPC {
        +ProbeGRPC(context.Context, string, Module, *prometheus.Registry, log.Logger) bool
    }

    class ResultHistory {
        +MaxResults: uint
        +results: List<Result>
        +Add(string, string, string, bool)
        +Get(int64) *Result
        +List() []Result
    }

    class Result {
        +Id: int64
        +ModuleName: string
        +Target: string
        +DebugOutput: string
        +Success: bool
    }

    class ScrapeLogger {
        +buffer: bytes.Buffer
        +Log(...interface{}) error
    }

    Main --> SafeConfig : uses
    Main --> ProberHandler : delegates /probe to
    Main --> ResultHistory : uses to display recent probes

    SafeConfig ..> Config : contains
    Config ..> Module : contains map of

    ProberHandler ..> Config : uses to get Module
    ProberHandler ..> Module : uses
    ProberHandler ..> ProbeFn : selects and calls
    ProberHandler ..> ResultHistory : stores results in
    ProberHandler ..> ScrapeLogger : uses for logging probe details

    ProbeHTTP ..|> ProbeFn : implements
    ProbeTCP ..|> ProbeFn : implements
    ProbeDNS ..|> ProbeFn : implements
    ProbeICMP ..|> ProbeFn : implements
    ProbeGRPC ..|> ProbeFn : implements
    
    ProbeHTTP ..> Module : uses HTTPProbe from
    ProbeTCP ..> Module : uses TCPProbe from
    ProbeDNS ..> Module : uses DNSProbe from
    ProbeICMP ..> Module : uses ICMPProbe from
    ProbeGRPC ..> Module : uses GRPCProbe from

    ResultHistory ..> Result : contains list of
```

### Diagram Legend and Notes:

*   **`Main`**: Represents the main package and its execution flow.
*   **`SafeConfig`, `Config`, `Module`**: Represent the configuration structure. `SafeConfig` provides safe access to `Config`, which holds multiple `Module` definitions.
*   **`ProberHandler`**: The logic that handles incoming `/probe` requests. It uses the `Config` to find the right `Module`, then selects and executes a `ProbeFn`.
*   **`ProbeFn`**: A function type representing any specific prober (HTTP, TCP, etc.). The diagram uses `<< (F,orchid) Function Type >>` to denote this. In Go, this is a function signature rather than a formal interface that classes explicitly implement.
*   **`ProbeHTTP`, `ProbeTCP`, etc.**: Concrete implementations of probers for different protocols. They match the `ProbeFn` signature.
*   **`ResultHistory`, `Result`**: Manage the storage and structure of recent probe outcomes.
*   **`ScrapeLogger`**: A specialized logger for capturing detailed logs during a probe execution, which can be later viewed for debugging.
*   **Relationships:**
    *   `-->`: Directed association (uses, interacts with).
    *   `..>`: Composition or aggregation (contains).
    *   `..|>`: Implementation / Conformance (a concrete prober function conforms to the `ProbeFn` signature).

This diagram aims to provide a high-level overview of the interactions rather than an exhaustive list of every function or variable.
The prober-specific configuration structs within `Module` (like `HTTPProbe`, `TCPProbe`) are implied but not fully expanded for brevity.
The `Probers` map in `prober/handler.go` which maps prober names (e.g., "http") to their respective `ProbeFn` functions (e.g., `ProbeHTTP`) is a key part of how `ProberHandler` dispatches to the correct prober.
