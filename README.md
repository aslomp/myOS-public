# myOS — Technical architecture

myOS is an independently developed operating system with one native kernel and multiple operating-system execution environments. The project explores how original Linux, Windows and Android user-space components can operate on shared native processes, memory, IPC, storage, networking and devices—without running three separate kernels.

**[Read the English technical documentation](docs/README.md)**

The documentation is written for technically experienced readers. It contains architecture explanations, cross-subsystem diagrams, security boundaries, observed capabilities and explicit limitations, without source-code listings.

> Documentation reflects the reviewed myOS master architecture and the development branch `claude/android-x86-64-native-624c9f` on 9 October 2026. A development implementation is not treated as a validated feature.

## Documentation objective

Provide a complete public-facing architectural explanation of myOS, illustrate relationships between systems with diagrams, and avoid disclosing source code. Keep observations, implementation state and design goals clearly distinguished.
