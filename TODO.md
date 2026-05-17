# TODO / Architecture Direction

Current state: s-ui goes DB → JSON template → write config.json → kill sing-box → restart → `box.New()`. Every CRUD restarts the entire core, dropping all connections.

## Goals

### 1. Drop the JSON config path

sing-box `option.*Options` structs can be constructed directly — no JSON marshal/unmarshal needed:

```
DB model → construct option.ShadowsocksInboundOptions{...} directly
         → inboundManager.Create(ctx, router, logger, tag, type, options)
         → Manager handles hot swap automatically (close old → 4-stage Start → wire into router)
```

`box.New()` only receives base config (log, route, dns). No inbounds/outbounds passed at startup.

### 2. Use sing-box's context DI for s-ui's own DI

sing-box uses `service.FromContext[T]()` / `service.ContextWith[T]()` for DI. s-ui follows suit:

```
ctx = service.ContextWith[FooService](ctx, fooServiceImpl)
// anywhere:
foo := service.FromContext[FooService](ctx)
```

No more manual `new` + global variables. s-ui's own service interfaces register into the same context.

### 3. Wrap a s-ui ServiceManager

```
s-ui ServiceManager
├── Track(dbRecordID → tag mapping)
├── ApplyInbound(dbRecord) → inboundManager.Create()
├── RemoveInbound(dbRecordID) → inboundManager.Remove()
├── ApplyOutbound(dbRecord) → outboundManager.Create()
├── RemoveOutbound(dbRecordID) → outboundManager.Remove()
└── ApplyRoute(rules) → router.Initialize()
```

CRUD API calls Apply/Remove directly. No restart signal returned.

### 4. Boot Box once

```
1. include.Context(ctx) → register all 6 Registries
2. service.ContextWith[FooService](ctx, ...) → register s-ui's own services
3. box.New(options) → base config only (log, route, dns)
4. box.Start()
5. Iterate DB → ApplyInbound / ApplyOutbound for each record
6. All subsequent CRUD → Apply / Remove directly, Box stays untouched
```

### 5. Layer separation

```
api/ (handler) → HTTP request/response only
service/      → s-ui business logic + sing-box Manager calls
adapter/      → s-ui interface definitions (registered into context)
db/           → GORM models + repository
```

Handler never touches DB. Service never touches HTTP. All interfaces go through context DI.

---

May or may not finish this, but the direction is set.
