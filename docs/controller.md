# Overview

# trainingjob_updater.go

## type TrainingJobController
```go
type TrainingJobController struct {
	// kubeCli is a standard kubernetes clientset
	kubeCli kubernetes.Interface
	// apiCli is the extension kubernetes clientset
	apiCli apiextensionsclient.Interface
	// paddleCli is a clientset for our own API group
	paddleCli paddleclientset.Interface

	trainingjobLister paddlelisters.TrainingJobLister
	trainingjobSynced cache.InformerSynced

	// jobtracker keeps a map from job full name to its updater
	jobtracker *sync.Map

	// workqueue is a rate limited work queue. This is used to queue work to be
	// processed instead of performing it as soon as a change happens.
	workqueue workqueue.RateLimitingInterface

	// recorder is an event recorder for recording Event resources to the
	// Kubernetes API.
	recorder record.EventRecorder

	// autoclean means whether to clean pods after termination.
	autoclean bool

	//restartLimit means pserver restart count reach to limit
	restartLimit int

	//outter meas if operator runs out of baidu
	outter bool
}
```
TOFILL

## func New
```go
func New(
	kubeCli kubernetes.Interface,
	apiCli apiextensionsclient.Interface,
	paddleCli paddleclientset.Interface,
	tjInformer paddleinformers.SharedInformerFactory,
	auto bool, restartLimit int, outter bool) *TrainingJobController
```
`New` constructs a `TrainingJobController` and sets up event handlers.


## func (*TrainingJobController) Run
```go
func (c *TrainingJobController) Run(threadiness int, maxLoadDesired float64, stopCh <-chan struct{}) error
```
`Run` will create CRD, sync informer caches, then start workers, a garbage collector, and an auto-scaler.
`Run` will block until stopCh is closed, at which point it will shutdown the workqueue and wait for workers to finish processing their current work items.


## func (*TrainingJobController) createCRD
```go
func (c *TrainingJobController) createCRD() error
```
NOTE: It is not using CRD generated by controller-gen, but a trivial one without validation.<br>
`createCRD` creates `TrainingJob` CRD on the cluster if it doesn't exist.


## func (*TrainingJobController) enqueue
```go
func (c *TrainingJobController) enqueue(obj interface{})
```
`enqueue` formats a namespace/name key for trainingjob `obj` and puts the key onto the work queue.


## func (*TrainingJobController) runWorker
```go
func (c *TrainingJobController) runWorker()
```
`runWorker` repeatedly runs [`processNextWorkItem`](#func-(*TrainingJobController)-processNextWorkItem) until it returns `false`.


## func (*TrainingJobController) processNextWorkItem
```go
func (c *TrainingJobController) processNextWorkItem() bool
```
FIXME: `forget` is ignored if `err != nil`.<br>
`processNextWorkItem` gets a trainingjob key from `c.workqueue` and [processes](#func-(*TrainingJobController)-syncHandler) it. It will add the key back to queue on error.



## func (*TrainingJobController) syncHandler
```go
func (c *TrainingJobController) syncHandler(key string) (bool, error)
```
FIXME: return values confusing.<br>
`syncHandler` fetches the jobUpdater for trainingjob `key` from `c.jobtracker`. If the jobUpdater doesn't exist, it will create one in `c.jobtracker`. If the job is deleted, it will delete the jobUpdater from `c.jobtracker`.
`syncHandler` then calls `Reconcile` on the jobUpdater if the job is not released. 
If the job is still running after `Reconcile`, `syncHandler` will add the job back to `c.workqueue`.



# garbage_collection.go

## type GarbageCollector
```go
type GarbageCollector struct {
	kubeCli           kubernetes.Interface
	trainingjobLister paddlelisters.TrainingJobLister
}
```

## func NewGarbageCollector
```go
func NewGarbageCollector(kubeCli kubernetes.Interface, trainingjobLister paddlelisters.TrainingJobLister) *GarbageCollector
```
`NewGarbageCollector` constructs and returns a new `GarbageCollector`.


## func (*GarbageCollector) CleanOrphans
```go
func (gc *GarbageCollector) CleanOrphans(d time.Duration)
```
`CleanOrphans` periodically finds and deletes replicasets, batchjobs, and pods that are not associated with a training job.
