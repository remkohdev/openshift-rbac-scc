# OpenShift RBAC with Security Context Constraints



This section is comprised of the following sections:

1. [Section 1: Authorization and User Permissions](#Section-1:-Authorization-and-User-Permissions)
1. [Section 2: Container Permissions and SCCs](#Section-2:-Container-Permissions-and-SCCs)

## Section 1: Authorization and User Permissions

1. It is recommended that you create a new project when deploying applications rather than working in the `default` project so let's create one.

    ```sh
    oc new-project rbac-project
    ```

    You will automatically target the new project with subsequent `oc` commands.

1. Each new project that is created contains 3 Service Accounts. Let's take a look at them:

    ```sh
    oc get sa
    ```

    You should see three listed:

    ```sh
    NAME        SECRETS   AGE
    builder     2         77m
    default     2         77m
    deployer    2         77m
    ```

    Each service account has it's own purpose:

    * builder: Used when running `build` pods
    * deployer: Used when running `deploy` pods
    * default: Default service account used if none is specified in the pod template

    We will actually see an example of a build pod later but it is typically used with the `oc new-app` and `oc start-build` commands.

1. Create a service account

    ```sh
    oc create sa helper
    ```

    In our fake example, this service account will be used to view the resources that are running on our cluster by querying the OpenShift API.

1. View the service accounts again:

    ```sh
    oc get sa
    ```

    You should now see:

    ```sh
    NAME       SECRETS   AGE
    builder    2         133m
    default    2         133m
    deployer   2         133m
    helper     2         132m
    ```

1. Let's authenticate with the OpenShift API to see what permissions our new Service Account has:

    1. Switch to your terminal and paste in `export TOKEN=` without entering yet.
    1. Open your OpenShift Cluster's console if you haven't already
    1. Click on `User Management` to expand the menu then click on `Service Accounts`
    1. In the list of service accounts, click on `helper`
    1. Scroll down to the area labeled `Secrets` and click on the secret labeled `helper-token`.
    1. On the secret page, scroll down and find the data field labeled `token`
    1. Click on the copy icon on the right of the `token` field.
    1. Switch back to your terminal and paste in the token. It should be a really long string of characters. It will look something like:

        ```sh
        export TOKEN=eyJhbGciOiJSUzI1NiIsImtpZCI6ImJrOUYzcUhSaGEwMk9mMXZMcWppWVpuZWdCQjBjcVM1d3ZJRk13ODluTGsifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJyYmFjLXByb2plY3QiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlY3JldC5uYW1lIjoiaGVscGVyLXRva2VuLXo5eDJ3Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImhlbHBlciIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImM4M2RiMDFiLWRiOTAtNGI2NS1iZWVkLWU4YzU2ZTM2ODkwNiIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpyYmFjLXByb2plY3Q6aGVscGVyIn0.JFw8vgrgVJhjAuP_8L5JVkPFPfK2w11Rzevi56qPlYvpUPaW4XJdVv3y3_8ieUWNGkWALO25--oj8Bqb8NYQ8mJuvh1D2vRL743iwJ8-fLah9KJO0wXe-OKNWNZAizSuz2EH3uJHpP6FqVJ8I-a0h015caS5VtrA16dSmShTer1i1JpSO8NxtUgnZLQtzfnkaclKyvvIFF8xcby83r8SWZGyGB7mxH7jQ5zdyw1cLwBgIXVXuSOpJA_4JmLMo3CZ2q8H-Nq20_3mD548z-fvv9vUxwgGSLTR1WGTPb2XDC5pEt3vF0oZULez406PHmF5Hd5fdLV2XGWXE1fHnnNoEA
        ```

    1. Then press enter

    Now that we have the token we need the server address of the cluster that we are authenticating with.

    1. In your terminal paste in `export SERVER=` without entering yet.
    1. Switch back to your OpenShift Console and click on your UserID at the top right of the page. It should start with `IAM#`.
    1. Click on `Copy login command`
    1. Then, click on `Display Token`. This will show your token and server address for your user account. All we need is the server address from this page.
    1. In the section titled `Log in with this token`, copy everything after the `--server=`. All we need is the address starting with `https` and ending in 5 numbers. This will be different for everybody.
    1. Go back to your terminal and paste in the server address so that the entire command reads something like:

        ```sh
        export SERVER=https://c109-e.us-east.containers.cloud.ibm.com:32204
        ```

    1. Now let's authenticate to the cluster as our `helper` service account:

        ```sh
        oc login --server=$SERVER --token=$TOKEN
        ```

1. As the new service account, try retrieving a list of all projects

    ```sh
    oc get projects
    ```

1. How about getting all resources?

    ```sh
    oc get all
    ```

1. Let's switch back to our own user account. Go back to the OpenShift login page and copy the entire login command this time and enter it into the terminal.

1. To give our new service account some permissions we will assign it the ClusterRole of `view`. This will allow `helper` to view the resources on the cluster without being able to modify anything. 

    First take a look at what permissions exactly we are giving `helper`:

    ```sh
    oc describe clusterrole view
    ```

    This looks like the ideal list of permissions for `helper` so let's grant him the ClusterRole.

    ```sh
    oc adm policy add-cluster-role-to-user view -z helper
    ```

1. Now let's deploy something for `helper` to view.

    ```bash
    oc new-app ruby~https://github.com/sclorg/ruby-ex.git
    ```

    This will create a simple ruby application for us. Remember when I mentioned that we would see build pods? Here is where they come into play. A build pod will build the sample app using the `builder` service account and will stop running when it is done and the application pods will then be available. There is a lot to go into regarding OpenShift builds but since that isn't the focus of this lab, we will leave that for another time.

1. Go to your terminal and login as `helper` again.

    ```sh
    oc login --server=$SERVER --token=$TOKEN
    ```

1. Now try running the following commands:

    ```sh
    oc get projects
    ```

    ```sh
    oc get pods -n rbac-project
    ```

    If our permissions are correct, `helper` can view the pods we just spun up with our user account.

    What if we try deploying something as `helper`?

    ```sh
    oc new-app ruby~https://github.com/sclorg/ruby-ex.git
    ```

    Notice in the `Creating Resources section` of the output that every resource failed to create due to the permissions that we have set.

In summary, we create a new service account for a `helper` that would be able to monitor workloads on our cluster and we assigned a limited ClusterRole of `view` that would ensure that it only has read access to our resources.

## Section 2: Container Permissions and SCCs

1. Log back in as your user account by copying the line from the OpenShift login page.

1. Let's create a new project for this section

    ```sh
    oc new-project scc-project
    ```

1. Examine the default `restricted` scc.

    ```sh
    oc describe scc restricted
    ```

1. Create a deployment with the default SCC.

    ```sh
    oc create -f RHELDeploy.yaml
    ```

1. Find the pod name that was just deployed

    ```sh
    oc get pods
    ```

1. Then, exec into the pod by running the following while replacing `{pod name}` with the actual pod name from the previous command:

    ```sh
    oc exec -it {pod name} -- bash
    ```

1. Once inside the pod, try the following:

    ```sh
    touch test.txt
    ```

    This should fail as RHEL has a lot of security in place to ensure that users do not have access to most locations.

    Now try this:

    ```sh
    touch tmp/test.txt
    ```

    This should work.

    Let's lock down the image even more by making the image read-only.

    Exit the pod by entering `exit`

1. Delete the deployment

    ```sh
    oc delete -f RHELDeploy.yaml
    ```

1. Let's create our own custom SCC

    ```sh
    oc create -f readonly-scc.yaml
    ```

    Let's examine the new SCC and see how it compares to `restricted`

    ```sh
    oc describe scc read-only
    ```

1. Create Service account
	
    ```sh
    oc create sa read-only
    ```

1. Now, create a Role that allows us to use the SCC with a service account
	1. oc create -f readonly-role.yaml

12. create role binding
	1. oc create -f readonly-rolebinding.yaml

13. Edit RHELDeploy.yaml
	1. remove # on line 17

14. create pod with service account in deployment
	1. oc create -f RHELDeploy.yaml

15. exec in and create files in root dir
	1. oc get pods
	2. oc exec -it {pod name} -- bash
	3. touch test.txt
	4. touch tmp/test.txt
    
16. success
