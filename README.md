# oshifted_hyrax
Hyrax in OpenShift

# Development using Open Shift
## Setup
Install Open Shift/Minishift (Reference: https://gist.github.com/h-parekh/bff04b97bdd8d2428771f9238d7a1e46):
1. `brew cask install minishift`
1. `brew cask install virtualbox` Note: May prompt for a password. Enter your normal login password.
1. Start open shift: `minishift start --vm-driver virtualbox`
1. Wait for a message like: `The server is accessible via web console at: https://x.x.x.x:8443`
1. Add the cli to your path: `eval $(minishift oc-env)` Note: Either add this to your shell profile, or make sure you run every time you open a new shell.
1. Test the cli: `oc version`

Create the project:
1. Clone this repository `git clone https://github.com/ndlib/oshifted_hyrax.git`
1. Login to open shift: `oc login`. Enter 'admin' for the username, and enter any password you like:
    ```
    Authentication required for https://192.168.99.100:8443 (openshift)
    Username: admin
    Password:
    Login successful.
    ```
    Note: You don't need to remember the password. You can always login as the admin user with any password. Just make sure to always use the same username
1. Create the project: `oc new-project hyrax`
1. Load the project configs: `oc create -f /<path_to_this_repo>/project.yaml`
1. Watch the build/deploy progress: `oc get builds --watch` and wait for a message like:
    ```console
    railsapp-1   Source    Git@76b3d46   Complete   3 minutes ago   3m5s
    ```
1. Alternatively, if you prefer more feedback, you can watch the logs for the running build:
    ```
    oc get builds --selector=app=railsapp | tail -n1 | awk '{ print $1 }' | xargs -I {} oc logs build/{} -f
    ```
    and wait for something like this:
    ```console
    ...
    Bundle complete! 8 Gemfile dependencies, 49 gems now installed.
    Gems in the group test were not installed.
    Bundled gems are installed into ./bundle.
    ---> Cleaning up unused ruby gems ...

    Pushing image 172.30.1.1:5000/hyrax/railsapp:latest ...
    Pushed 6/9 layers, 67% complete
    Pushed 7/9 layers, 78% complete
    Pushed 8/9 layers, 89% complete
    Pushed 9/9 layers, 100% complete
    Push successful
    ```
1. Test with a browser at: http://hyrax-application-route-hyrax.192.168.99.100.nip.io/

## Iterating on the Application Code
Since the containers are not reading the files on your host OS, there are two things that need to be done before the running instance will reflect changes you make to the application code.

### 1. Sync the files to the container
For dynamically reloaded files, such as controllers, views, models, etc., running a constant rsync that will copy files from your host os to the container should be sufficient.

From the root of the project directory:
`scripts/ocsync railsapp .`

Example:
```
lib-2001:oshifted_api jgondron$ scripts/ocsync railsapp .
building file list ... done
...
```

Changing a file should trigger the rsync (within a few seconds) and you should be able to refresh the page to see the change.

### 2. Rebuild the image from your local files
For files that require a rebuild or a restart of rails, such as secrets, Gemfile, etc., you will need to rebuild the image. You can do this without having to push to the repo by running a build using the local directory.

From the root of the project directory:
`scripts/ocbuild railsapp /<path_to_hyrax_repo>`

Wait for the build to complete, then resume the rsync.

Example:
```
lib-2001:oshifted_api jgondron$ scripts/ocbuild railsapp .
Uploading directory "." as binary input for the build ...
build "railsapp-5" started
NAME         TYPE      FROM          STATUS     STARTED        DURATION
railsapp-1   Source    Git@76b3d46   Complete   24 hours ago   1m28s
railsapp-2   Source    Binary@76b3d46   Complete   24 hours ago   1m56s
railsapp-3   Source    Binary@76b3d46   Complete   24 hours ago   1m52s
railsapp-4   Source    Binary@76b3d46   Complete   18 minutes ago   1m23s
railsapp-5   Source    Binary@76b3d46   Running   15 minutes ago
railsapp-5   Source    Binary@76b3d46   Complete   15 minutes ago   1m52s
^C
lib-2001:oshifted_api jgondron$ scripts/ocsync railsapp .
building file list ... done
...
```

## Debugging
Debugging requires running a remote debug client. The ocdebug script will start a terminal within the running pod for you and begin a debug client.

From the root of the project directory:
`scripts/ocdebug railsapp`

Example:
```
lib-2001:oshifted_api jgondron$ scripts/ocdebug railsapp
Connecting to byebug server...
Connected.
```
Once this is connected, you should be able to set a breakpoint with byebug and interact with it the same as if you were developing locally. For example, go to http://railsapi-oshifted-api.192.168.99.100.nip.io/debug and you should see:
```
[5, 14] in /opt/app-root/src/app/controllers/echo_controller.rb
    5:       secret: Rails.application.secrets.some_other_secret,
    6:       database: Rails.application.config.database_configuration['development']['some_other_config']
    7:     }
    8:   end
    9:
   10:   def debug
   11:     byebug
=> 12:     render json: {}
   13:   end
   14: end
(byebug)
```

## Iterating on the Open Shift configuration
Edit the running configuration until happy, then it can be dumped via:
```
oc export service,route,imagestream,buildconfig,deploymentconfig -o yaml > project.yaml
```
Commit the changes to this file to the repo. To apply those changes on other developer machines, use `oc apply`.

If something has changed, ie not been removed, just an apply with an overwrite seems to work:
```
oc apply --overwrite=true -f project.yaml
```

If something has been removed, you have to also force a prune:
```
oc apply --all --overwrite=true --prune=true -f project.yaml
```

Note: These are very broad strokes. Not sure if it’s safe to always use prune for what we are doing.
