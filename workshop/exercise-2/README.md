## Setup: Clone the Class Repo

### Clone the Class repo
During the course of this lab we'll deploy a variety of sample applications and apply various `yaml` files to implement our desired behavior in Kubernetes. Cloning this lab will make it easy to apply the yaml without

1. Clone the git repository:

    ```
    git clone https://github.com/kitch/kube-operations101.git
    ```
2. Change directories to the fib-knative folder.

    ```
    cd fib-knative
    ```

Congratulations! You've set up some required credentials that the Knative build process will use to have access to push to your container registry. In the next exercise, you will install Knative onto the cluster so that you can build and deploy this app. The goal of this exercise was to set up some required credentials.


Continue on to [exercise 3](../exercise-3/README.md).
