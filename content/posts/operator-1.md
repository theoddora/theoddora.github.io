---
date: '2025-10-09T18:15:00+03:00'
draft: false
title: 'Writing Your First Kubernetes Operator - The Definitions'
tags: ["k8s", "kubernetes", "operator", "controller-runtime"]
---
 
If you're like me and just starting your Kubernetes journey, you'll likely need to write an Operator to manage your custom resources. Kubernetes offers several approaches to building operators, including:
- `controller-runtime` — a Go framework for building controllers and operators
- `kubebuilder` — a CLI tool that scaffolds an operator project using `controller-runtime`

#### Controller vs Operator

> **Controller** = logic that keeps the actual state aligned with the desired state.

> **Operator** = a controller that manages an application like a human operator would, using **Custom Resources**.  
> In short: **Operator = Controller + Custom Resource (CRD) + Domain Knowledge**


In this guide, we will focus on writing an operator using controller runtime. 

But first, it is important to have a conceptual idea of what we are using to achieve our goal. 

---

## API Machinery and Controller Runtime

We will add conceptual knowledge and main definitions of our building blocks:
- API Machinery - `"k8s.io/apimachinery"`
- Controller Runtime - `"sigs.k8s.io/controller-runtime"`

### Architecture
On the bottom layer, we have the **Kubernetes API server** and **etcd**.  
Building on top of that is **API Machinery**, and on top of that, we have **controller-runtime**.

```
+---------------------------------------------------------+
| controller-runtime (framework for building controllers)  |
|   ├── Manager                                            |
|   ├── Controller                                         |
|   ├── Reconciler                                         |
|   ├── Client (wraps API Machinery client)                |
+---------------------------------------------------------+
| Kubernetes API Machinery (low-level API tools)           |
|   ├── runtime.Scheme, Object, Serializer                 |
|   ├── GroupVersionKind, GroupResource                    |
|   ├── DeepCopy, Conversion, Meta types                   |
+---------------------------------------------------------+
| Kubernetes API Server & etcd                             |
+---------------------------------------------------------+
```

## API Machinery
API Machinery is the **foundation** that defines _what_ Kubernetes objects are and _how_ they are encoded and handled. It tells Kubernetes how to understand, version, and serialize objects.

This is the **core library** of Kubernetes - the low-level building blocks that make the Kubernetes API function.

It handles all the "API plumbing" such as:
- Serialization and deserialization (JSON <-> Go structs)
- Type registration (`Scheme`, `GroupVersionKind`)
- Metadata (`ObjectMeta`, `TypeMeta`)
- Versioning and conversion logic
- Deep copying and object interfaces
- REST resource identification (`GroupVersionResource`, etc.)

Outlining the main types our Operator will use
- `TypeMeta` - Serialization, deserialization, API routing
- `ObjectMeta` - Uniqueness, lifecycle, metadata tracking
- `GroupVersionResource` - Kubernetes API resources are fully identified by a **group-version-resource (GVR)** triple: example - group:apps, version: v1, resource: deployment

**Key packages of API Machinery:**
- `runtime` - defines `runtime.Object`, `Scheme`, and type registration.
- `schema` - defines `GroupVersion`, `GroupKind`, `GroupResource`, etc.
- `metav1` - defines metadata structs like `ObjectMeta`, `ListMeta`.
- `util/runtime`, `util/validation` - utility helpers.
- `fields`, `labels`, etc. - for selectors.

### What is a `Schema` actually?

A **Schema** (represented by `runtime.Scheme` in Go) defines **how objects of different types are recognized, versioned, and converted**. It acts as a **registry** of all known Kubernetes types in our controller.

When we register a type with a Scheme, we are actually saying:
- "This Go struct represents a Kubernetes resource."
- "Here's its group, version, and kind (GVK)."
- "Here's how to convert it between versions if needed."

The Scheme is crucial because it allows our operator to:
1. Encode and decode objects to/from JSON or YAML.
2. Identify an object's type when reading from the API or cache.
3. Support version conversions when our CRD evolves.

Without a Scheme, our controller wouldn't know how to handle our custom resources properly, and most controller-runtime operations like `Get`, `List`, or `Create` would fail.
## Controller runtime

Key components and architecture of the controller-runtime:

```
       ┌──────────────────────────────────────────┐
       │                Manager                   │
       │  - sets up cache, client, recorder       │
       │  - starts controllers                    │
       └──────────────┬───────────────────────────┘
                      │
                      ▼
             ┌─────────────────────┐
             │     Controller      │
             │ - watches resources │
             │ - enqueues requests │
             └──────────┬──────────┘
                        │
                        ▼
             ┌─────────────────────┐
             │     Reconciler      │
             │ - gets objects      │
             │ - compares desired  │
             │ - updates cluster   │
             │ - reports status    │
             └─────────────────────┘

```

The controller runtime is a **high-level framework** for building **controllers and operators** in Go - built _on top of API Machinery_.

**It provides:**
- A **Manager** (lifecycle & dependency injection)
- **Controllers** and **Reconcilers**
- A unified **Client** (reads from cache, writes to API) - a **Kubernetes API client**
- Shared **Caches** (informers under the hood)
- Support for **webhooks**, **leader election**, and **metrics**

#### Client
Unified Client (reads from cache, writes to API)

> "Unified" = one consistent interface for both **cached reads** and **direct writes**, instead of two separate systems.

When we create a Manager, it will automatically build a `Client` and inject it into our controllers. 
```go
mgr, _ := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
    Scheme: scheme,
})

myReconciler := &controllers.MyReconciler{
    Client: mgr.GetClient(),  // <-- here's the client
}
```

So, when our controller does something like:
```go
pod := &corev1.Pod{} err := r.Client.Get(ctx, client.ObjectKey{Name: "mypod", Namespace: "default"}, pod)
```
it's using the **Client** to _get_ the Pod object - usually from the **cache** (for speed).

Or when it does:
```go
r.Client.Update(ctx, pod)
```
it's using the **Client** to _write_ the updated Pod back to the **Kubernetes API server**.

#### Manager
Kubernetes controllers need a **shared environment**:
- One `Client` that uses the shared informer cache
- One `Scheme` for object type conversions
- One `Recorder` for emitting events
- One `Config` (the REST client configuration)
- One `APIReader` for uncached reads

So the **Manager** centralizes them, then **injects** them into every controller or runnable that needs them.

Dependency injection means we don't create our dependencies ourselves, we receive them from something else that manages their lifecycle. Instead of our controller manually constructing clients, caches, recorders, etc., the Manager creates them once, shares them across all controllers, and injects them into each component.

In Go, controller-runtime doesn't use a reflection-based DI framework like Java's Spring - it uses **struct fields** and **interfaces** to achieve DI _statically_.

Each controller's Reconciler usually looks like this:
```go
type MyReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}
```

Then in `main.go`, it wires it up like this:
```go
reconciler := &controllers.MyReconciler{
    Client: mgr.GetClient(),
    Scheme: mgr.GetScheme(),
}

if err := reconciler.SetupWithManager(mgr); err != nil {
    log.Fatal(err)
}
```

#### Caches (informers?)
We know what a cache is, but what is an informer and why do we consider them the same in the context of controller-runtime? 

**Cache** in controller-runtime is essentially an in-memory store of Kubernetes objects. It keeps copies of resources (Pods, Deployments, CRs, etc.) so our controllers can read quickly without hitting the API server every time.

**Informer** is a mechanism that feeds the cache. It "informs" our controller about changes in resources. Specifically, it:
- Watches the Kubernetes API for a particular resource type.
- Keeps the cache updated with the latest state.
- Triggers events (Add, Update, Delete) so our controller can react.

In controller-runtime, caches are backed by informers, which watch Kubernetes resources and keep local copies up-to-date. This allows controllers to read from the cache for speed instead of querying the API server directly.

```
Kubernetes API Server
         │
         ▼
      Informer
         │  (event)
         ▼
       Queue ──> Reconciler
         │          │
         │          ▼
         │      Reconcile(obj)
         │
         └─ retries / backoff if needed
```
Due to the informer, our controller will know that something has changed without constantly polling the KAPI.

#### Reconciler Queue
The queue is an internal *work queue* that holds "reconciliation requests" — it references to Kubernetes objects that need to be reconciled. This is a thread-safe queue for storing items (usually `ctrl.Requests`). When an event occurs (create, update, delete) that affects a watched object, the controller enqueues a request for the Reconciler to process. The Reconciler then pulls items from the queue one at a time (or in parallel) and runs the `Reconcile()` function.

The Queue is:
- Event-driven: manually loop over all objects; the queue is populated by informers whenever a relevant change happens.
- Rate-limiting: it can retry failed reconciliations using backoff, preventing hot loops when errors occur.
- Deduplication: if multiple events happen for the same object before it’s processed, the queue usually coalesces them into a single reconciliation request to avoid redundant work.
- Order isn't guaranteed: Items are generally processed in FIFO order, but because of retries and concurrency, exact ordering isn't guaranteed.

#### Reconciler
A **Reconciler** is the core of a controller - the logic that defines **what to do** when a Kubernetes object changes.
It implements:
```go
type Reconciler interface {
    Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error)
}
```

Lets check the signature:
`ctx context.Context` - Used to handle timeouts, cancellations, and deadlines (especially during shutdown).

`req ctrl.Request` - Carries metadata about **which object** needs reconciling:
```go
type Request struct {
    NamespacedName types.NamespacedName // {Namespace, Name}
}
```
This identifies a _single object instance_ (like `"default/my-app"`).

Returns -> `(ctrl.Result, error)`

| Return Value                                  | Meaning                                    |
| --------------------------------------------- | ------------------------------------------ |
| `ctrl.Result{}`                               | Reconciliation successful - don't requeue. |
| `ctrl.Result{Requeue: true}`                  | Re-run immediately.                        |
| `ctrl.Result{RequeueAfter: 10 * time.Second}` | Re-run after some delay.                   |
| `error != nil`                                | Error - requeue with backoff.              |

This mechanism lets us control _how often_ reconciliation runs and how errors are retried.

The Reconciler doesn't run constantly - it's **event-driven**.
1. The **primary resource** changes (e.g., a CRD or Deployment).
2. A **dependent resource** changes (e.g., Pod, Service, etc.).
3. A **manual requeue** or `RequeueAfter` is triggered.
4. The system detects a transient error and retries.

So `Reconcile()` is not a loop we write - it's a _callback_ triggered by events.

Example:
```go
func (r *MyReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // Read the object from cache
    myObj := &myv1.MyResource{}
    if err := r.Client.Get(ctx, req.NamespacedName, myObj); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Business logic: ensure a Deployment exists for this resource
    desired := newDeploymentForMyResource(myObj)
    if err := ctrl.SetControllerReference(myObj, desired, r.Scheme); err != nil {
        return ctrl.Result{}, err
    }

    // Apply the desired Deployment
    if err := r.Client.Create(ctx, desired); err != nil && !apierrors.IsAlreadyExists(err) {
        return ctrl.Result{}, err
    }

    return ctrl.Result{}, nil
}
```


With this we are all set up in our journey to write our operator.
