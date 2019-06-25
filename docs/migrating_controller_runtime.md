# Migrating to Controller-Runtime

**TL;DR**: From version `v1.12.alpha.3` on, we migrated controller framework to [controller-runtime](https://github.com/kubernetes-sigs/controller-runtime)
which is for now actively maintained by trusted groups. If this is your first time trying out the project, you don't need
to continue reading. Otherwise if your project is generated before the version, you will hit some backward-compatibility
issues on upgrading (apologizes we didn't make the feature opt-in b/c integrating controller-runtime literally involves
a through out change in the codebase) and the doc will show you a path to migrating to the new framework w/o hiccups.


### Break-changes in the project

- The controller generator now has been removed from `apiserver-boot`, So we will not generate source files `zz_generated.api.register.go`
under `pkg/controller/...` packages any more.
- The new controller template integrated controller-runtime will only be generated for one time when you create the resource.
- The new controller template now contains not only a dummy controller but also local testing environment. So when you run
the tests, it'll bootstrap an local kubernetes cluster and also install an aggregated apiserver, expecting a 10s-ish time cost
for setting up everything. Note that the aggregated apiserver *has to* be running on `443` port which may acquire additional
priviledges. In this case, consider running `sudo env "GOPATH=$GOPATH" go test ./...` for privilege escalation.

### New to Controller-Runtime?

The point of controller-runtime is to construct a unified controller framework helping developers get rid of troubles from
copy/pasting boiler-plates codes when developing a kubernetes controller. The framework  implements a Kubernetes API by
responding to events (object Create, Update, Delete) and ensuring that the state specified in the Spec of the object
matches the state of the system. That is called a reconcile. If they do not match, the Controller will create / update / delete
objects as needed to make them match.


### What if I don't feel like exerting Controller-Runtime?

Upgrading vendor dependencies for the project will **always** assures backward-compatibility but the behavior of `apiserver-boot`
isn't. Vendor updates shall never break anything so that users can track the latest kubernetes w/o conflicts. However, as
for the code re-generation, you'll have to use the `v1.12.alpha.2` version of `apiserver-boot` so that codes won't mess up.

### How to migrate?

A complete process of migrating your existing projects to controller-runtime will basically consists of 5 steps:

1. Download the latest binaries of `apiserver-builder-alpha` and the version should be at least prior to `v1.12.alpha.3`
2. Run `apiserver-boot init repo --domain=<your domain>`, and the command is supposed to generated a new main source file
under `cmd/manager`. Note that the package name is changed on purpose so that it won't conflict w/ the existing sources.
3. Then we will move no to migrate the actual controller packages. e.g. for migrating a resource named `foos` in group `testing`
and version `v1alpha1`, run `apiserver-boot create resource --group=testing --version v1alpha1 --resource foos --kind Foo`
and it will generate the templated controller using controller-runtime under `pkg/controller` incrementally. Note that
all the existing controller sources **will never** be over-written in this step b/c the re-generated file names are different
from the legacy ones:
    - **Legacy controller source file names**: `controller.go`, `controller_test.go`, `foo_suite_test.go`.
    - **New controller source file names**: `foo_controller.go`, `foo_controller_suite_test.go`, `foo_controller_test.go`.
4. After learning about how controller-runtime works, migrate the existing controller's logic and also it's tests to the
new source files.
5. Clean-ups. Delete the legacy controller files as mentioned above and don't forget to delete `zz_generated.api/register.go`
from the package.
6. Make sure the tests happy. And file an issue if you encounter any issues.



