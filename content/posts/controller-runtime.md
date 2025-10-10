---
date: '2025-10-09T18:15:00+03:00'
draft: false
title: 'Demystifying Kubernetes `controller-runtime`'
tags: ["k8s", "kubernetes", "operator", "controller-runtime"]
toc: true
---
 
If you're like me and just starting your Kubernetes journey, you'll likely need to write an [Operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) to manage your [custom resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/). Kubernetes offers a high-level Go framework called [`controller-runtime`](https://github.com/kubernetes-sigs/controller-runtime) for building such operators; frameworks like [Kubebuilder](https://book.kubebuilder.io/)  are built on top of it. What do I mean by custom resources? Well, anything that you would create which is not part of the Kubernetes well-known built-in resources - Pods, Deployments, ReplicaSets, etc. An example might be you adding a new custom `Backup` resource whose main purpose is to do a scheduled cron job to upload backups to a desired target storage solution. 

#### Controller vs Operator

> **Controller** = logic that keeps the actual state aligned with the desired state.

> **Operator** = Controller + Custom Resource (CRD) + Domain Knowledge

In this guide, we will focus on understanding the components behind `controller-runtime` and what capabilities it has to offer. Where does it fit the whole kubernetes picture? What capabilities does it provide of write operators?

---

## `controller-runtime` dependencies

What does the controller-runtime depend on? The main thing that concerns us is the API Machinery - `"k8s.io/apimachinery"`. We cover the relevant parts of API Machinery later, so you understand the types used without losing focus on the main topic.

## Controller runtime

The controller runtime is a **high-level framework** for building **controllers and operators** in Go - built _on top of API Machinery_. It provides:
- A **Manager** (lifecycle & dependency injection)
- **Controllers** and **Reconcilers**
- A unified **Client** (reads from cache, writes to API) - a **Kubernetes API client**
- Shared **Caches** (informers under the hood)
- Support for **webhooks**, **leader election**, and **metrics**

![](/resources/controller-runtime-arch.png)
*Flow of controller-runtime event processing*

Let's dive deep in each component.

#### Manager
Kubernetes controllers need a **shared environment**. As we need a consistent and up-to-date view of the cluster, a shared environment means that multiple controllers (or multiple parts of the same controller) operate within a common set of shared parts — like caches, clients, schemes, and configuration.
- One [`Client`](#client) that uses the shared informer cache
- One [`Scheme`](#what-is-a-schema-actually) for object type conversions
- One `Recorder` for observability - used to generate Kubernetes events
- One `Config` (the REST client configuration)
- One `APIReader` for uncached reads
- One `metrics` endpoint which exposes metrics by default on `/metrics` (used by Prometheus).
- One `healthz/readyz` probe - endpoints to integrate with Kubernetes liveness/readiness checks.
- One (optional) `Leader Election` so that only one instance actively reconciles at a time

The **Manager** centralizes them, then **injects** them into every controller or runnable that needs them.

Dependency injection means we don't create our dependencies ourselves, we receive them from something else that manages their lifecycle. Instead of the controllers manually constructing clients, caches, recorders, etc., the Manager creates them once, shares them across all controllers, and injects them into each component.

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

#### Client
We need a way to interact with our cluster. The `k8s.io/client-go` package provides the tools to create a client. The `controller-runtime` leverages it to create a more robust **unified** client.

> "Unified" = single interface for cached reads and direct writes, avoiding the need to use separate client-go constructs.

When we create a Manager, it will automatically build a `Client` and inject it into our controllers. 
```go
mgr, _ := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
    Scheme: scheme,
})

myReconciler := &controllers.MyReconciler{
    Client: mgr.GetClient(),  // <-- the client is available, can be used in other controllers as well
}
```

Read flow:
```go
pod := &corev1.Pod{} err := r.Client.Get(ctx, client.ObjectKey{Name: "mypod", Namespace: "default"}, pod)
```
it's using the **Client** to _get_ the Pod object - usually from the cache.

Write flow:
```go
r.Client.Update(ctx, pod)
```
it's using the **Client** to _write_ the updated Pod back to the **Kubernetes API server**.


#### Cache/Informer

**Cache** in `controller-runtime` is an in-memory store of Kubernetes objects. It keeps copies of resources (Pods, Deployments, CRs, etc.) so our controllers can read quickly without hitting the API server every time.

In `controller-runtime`, caches are backed by informers, which watch Kubernetes resources and keep local copies up-to-date. This allows controllers to read from the cache for speed instead of querying the API server directly. 

**Informer** offer an elegant solution, acting as a smart filter between the controller and the Kubernetes API server. It "informs" our controller about changes in resources. Specifically, it:
- Watches the Kubernetes API for a particular resource type.
- Keeps the cache updated with the latest state.
- Triggers events (Add, Update, Delete) on *any* resource change, so our controller can react.

Due to the informer, our controller will know that something has changed without constantly polling the KAPI.

Behind the scenes, here is what the informer client does ([ref](https://render.com/blog/kubernetes-informers)):

1. On initialization, the Informer sends a `LIST` request to fetch all related resources in the cluster - i.e. Pods. Because the Informer's cache starts out empty, all returned Pods are treated as "new" and passed to AddFunc.
   - This response includes a resourceVersion marker, which is used in the next step.
   - Importantly, the Informer stores returned Pods in its in-memory cache, which is kept up-to-date using events from the `WATCH` below.
2. The Informer then sends a `WATCH` request, which *subscribes* to all Pod updates that happen after the resourceVersion marker.
   - Pod creations are passed to AddFunc, deletions to DeleteFunc, and modifications to UpdateFunc.

#### Reconciler WorkQueue
The queue is an internal *work queue* that holds "reconciliation requests" - it references to Kubernetes objects that need to be reconciled. This is a thread-safe queue for storing items (usually `ctrl.Requests`). When an event occurs (create, update, delete) that affects a watched object, the controller enqueues a request for the Reconciler to process. The Reconciler then pulls items from the queue one at a time (or in parallel) and runs the `Reconcile()` function.

The Queue properties:
- Event-driven: the queue is populated by informers whenever a relevant change happens.
- Rate-limiting: prevents overloading the controller.
- Error Backoff: it can retry failed reconciliations using backoff, preventing "hot" loops when errors occur.
- Deduplication: if multiple events happen for the same object before it’s processed, the queue usually coalesces them into a single reconciliation request.
- Order isn't guaranteed: Items are generally processed in FIFO order, but because of retries and concurrency, exact ordering isn't guaranteed.

#### Reconciler
A **Reconciler** is the core of a controller - the logic that defines **what to do** when a watched Kubernetes object changes.
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
    NamespacedName types.NamespacedName // object represented by {namespace str, name str}
}
```
This identifies a _single object instance_ (like `"backup-ns/my-backup"`).

Returns -> `(ctrl.Result, error)`

| Return Value                                  | Meaning                                    |
| --------------------------------------------- | ------------------------------------------ |
| `ctrl.Result{}`                               | Reconciliation successful - don't requeue. |
| `ctrl.Result{Requeue: true}`                  | (deprecated) If the error is nil and result.RequeueAfter is zero and result.Requeue is true, the request will be requeued using exponential backoff.|
| `ctrl.Result{RequeueAfter: 10 * time.Second}` | Re-run after the specified delay.          |
| `error != nil`                                | Error - requeue with backoff mechanism (skip if terminal error). Ignores ctrl.Result.|

This mechanism lets us control _how often_ reconciliation runs and how errors are retried.

So `Reconcile()` is not a loop we write - it's a _callback_ triggered by events - it's **event-driven**.
1. The **primary resource** changes (e.g., a CRD or Deployment).
2. A **dependent resource** changes (e.g., Pod, Service, etc.).
3. A **manual requeue** or `RequeueAfter` is triggered.
4. The system detects a transient error and retries.


Example in code:
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

## API Machinery
API Machinery is the **foundation** that defines _what_ Kubernetes objects are and _how_ they are encoded and handled. It tells Kubernetes how to understand, version, and serialize objects.

This is the **core library** of Kubernetes - the low-level building blocks that make the Kubernetes API function.

It handles all the important tasks such as:
- Serialization and deserialization (JSON <-> Go structs)
- Type registration (`Scheme`, `GroupVersionKind`)
- Metadata (`ObjectMeta`, `TypeMeta`)
- Versioning and conversion logic
- Deep copying and object interfaces
- REST resource identification (`GroupVersionResource`, etc.)

When we define a Kubernetes object, we specify its `TypeMeta` (Kind and APIVersion - needed for Serialization, deserialization, API routing).

Every Kubernetes object also has an `ObjectMeta`, to define its uniqueness, leveraged by the object lifecycle and metadata tracking.

The full identifier of a Kubernetes API object is the `GroupVersionResource` commonly written as group/version/resource (GVR) triple: example - group:apps, version: v1, resource: deployment.

**Key packages of API Machinery:**
- `runtime` - defines `runtime.Object`, `Scheme`, and type registration.
- `schema` - defines `GroupVersion`, `GroupKind`, `GroupResource`, etc.
- `metav1` - defines metadata structs like `ObjectMeta`, `ListMeta`, `TypeMeta`.
- `util/runtime`, `util/validation` - utility helpers.

#### What is a `Schema` actually?

A **Schema** (represented by `runtime.Scheme` in Go) defines **how objects of different types are recognized, versioned, and converted**. It acts as a **registry** of all known Kubernetes types in our controller.

When we register a type with a Scheme, we are actually saying:
- "This Go struct represents a Kubernetes resource."
- "Here's its group, version, and kind (GVK)."
- "Here's how to convert it between versions if needed."

The Scheme is crucial because it allows our operators to:
1. Encode and decode objects to/from JSON or YAML.
2. Identify an object's type when reading from the API or cache.
3. Support version conversions when our CRD evolves.

Without a Scheme, our controller wouldn't know how to handle our custom resources properly, and most controller-runtime operations like `Get`, `List`, or `Create` would fail.

## Useful Reads
- [Demystifying Kubernetes Informers](https://medium.com/@jeevanragula/demystifying-kubernetes-informer-streamlining-event-driven-workflows-955285166993)
- [Kubernetes Informers are so easy... to misuse!](https://render.com/blog/kubernetes-informers)
