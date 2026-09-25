# Dependency Injection Graph

**Status:** NORMATIVE.

Hilt is used only in Android/application modules. Pure JVM protocol/domain modules expose constructors and interfaces.

## Singleton scope

- RuntimePackLoader
- RuntimePackValidator
- active RuntimePackStore
- AutoDiagDatabase and DAOs
- TransportFactory registry
- AdapterFactory registry
- AutoDiagLogger
- TraceRecorder
- DispatcherProvider
- AutoDiagClock

## VehicleSession custom scope

A custom `@VehicleSessionScoped` component is created after physical adapter connection and destroyed on disconnect.

Scoped objects:
- selected Transport
- VehicleAdapter
- DiagnosticExecutor session coordinator
- EcuDiscoveryEngine
- EcuIdentityReader
- VehicleDetector
- VariantSelector
- LiveDataScheduler
- WriteTransactionEngine
- ServiceRuntime
- session Mutex
- session correlation/trace context

Protocol `DiagnosticChannel` objects are shorter-lived operation resources and are explicitly closed.

## ViewModel scope

ViewModels depend only on use cases and immutable UI mappers. No Transport, VehicleAdapter, DiagnosticChannel or DAO is injected directly into ViewModels.

## Forbidden

- protocol objects as process singletons;
- Activity/Context injected into JVM core modules;
- Room DAOs outside storage/usecase orchestration boundaries.
