Production Deployment
=====================

Example how to deploy Saferwall for single-instance production environment.

Prerequisites
-------------

Make sure you have deployed the following services:

* MinIO
* Couchbase
* NSQ

Clone repositories
------------------

.. code-block:: bash

    mkdir ~/extra2000
    cd ~/extra2000
    git clone https://github.com/extra2000/saferwall-podman.git
    git clone --recursive https://github.com/saferwall/saferwall.git saferwall-podman/src/saferwall
    git clone https://github.com/extra2000/saferwall-ui.git saferwall-podman/src/saferwall-ui
    git clone https://github.com/saferwall/saferwall-api.git saferwall-podman/src/saferwall-api
    git clone https://github.com/saferwall/pe.git saferwall-podman/src/saferwall-pe

Pull MultiAV Images
--------------------

.. code-block:: bash

    podman image pull docker.io/saferwall/goclamav:latest
    podman image pull docker.io/saferwall/gocomodo:latest
    podman image pull docker.io/saferwall/goavira:latest

Pull Saferwall Images
---------------------

.. code-block:: bash

    podman image pull docker.io/saferwall/webapis:latest
    podman image pull docker.io/saferwall/orchestrator:latest
    podman image pull docker.io/saferwall/aggregator:latest
    podman image pull docker.io/saferwall/postprocessor:latest
    podman image pull docker.io/saferwall/pe:latest
    podman image pull docker.io/saferwall/gometa:latest

Build Saferwall Images
----------------------

From the project root directory, ``cd`` into ``src/saferwall-ui/`` and then build:

.. code-block:: bash

    cd src/saferwall-ui/
    podman build -t saferwall/ui .

Configure Couchbase
-------------------

Go to http://localhost:8091, create ``sfw`` buckets with the following configurations:

* Bucket Type: Couchbase

Create the following user:

* Username: ``saferwall-system``
* Password: ``abcde12345``
* Allow application access to ``sfw``

Configure MinIO
---------------

Create Saferwall Buckets
~~~~~~~~~~~~~~~~~~~~~~~~

Create Elasticsearch bucket named ``saferwall-samples`` and ``saferwall-images``.

Create IAM Policy
~~~~~~~~~~~~~~~~~

Create a policy named ``saferwall-system-rw``:

.. code-block:: json

    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "s3:ListBucket",
                    "s3:GetBucketLocation",
                    "s3:ListBucketMultipartUploads",
                    "s3:ListBucketVersions"
                ],
                "Resource": [
                    "arn:aws:s3:::saferwall-samples/*",
                    "arn:aws:s3:::saferwall-images/*"
                ]
            },
            {
                "Effect": "Allow",
                "Action": [
                    "s3:GetObject",
                    "s3:PutObject",
                    "s3:DeleteObject",
                    "s3:AbortMultipartUpload",
                    "s3:ListMultipartUploadParts"
                ],
                "Resource": [
                    "arn:aws:s3:::saferwall-samples/*",
                    "arn:aws:s3:::saferwall-images/*"
                ]
            }
        ]
    }

Create MinIO User for Managing Saferwall System Account
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Create a new user with the following options:

* Access Key: ``saferwall-system-admin``
* Secret Key: ``abcde12345``
* Assign Policies: ``saferwall-system-rw``

.. warning::

    User ``saferwall-system-admin`` will be strictly used for login MinIO console and manage service accounts. DO NOT use it on Saferwall or any services. Instead, use it's service account.

Create MinIO Service Account for Saferwall System
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Login as ``saferwall-system-admin`` and then create a new service account:

* Customize Credentials: ``OFF``
* Restrict with policy: ``OFF``

.. note::

    Make sure to remember the service account access key and secret key. These keys will be used by Saferwall.

Deploy Saferwall Webapis
------------------------

From the project root directory, ``cd`` into ``deployment/production/webapis``:

.. code-block:: bash

    cd deployment/production/webapis

Create config files and fix permissions:

.. code-block:: bash

    cp -v configmaps/saferwall-webapis.yaml{.example,}
    cp -v configs/local.toml{.example,}
    chmod og+r configs/local.toml

Create pod file:

.. code-block:: bash

    cp -v saferwall-webapis-pod.yaml{.example,}

For SELinux platform, label the following files to allow to be mounted into container:

.. code-block:: bash

    chcon -R -v -t container_file_t ./configs

Load SELinux security policy:

.. code-block:: bash

    sudo semodule -i selinux/saferwall_webapis.cil /usr/share/udica/templates/base_container.cil

Verify that the SELinux module exists:

.. code-block:: bash

    sudo semodule --list | grep -e "saferwall_webapis"

Deploy Saferwall Webapis:

.. code-block:: bash

    podman play kube --configmap configmaps/saferwall-webapis.yaml --seccomp-profile-root ./seccomp saferwall-webapis-pod.yaml

Create systemd files to run at startup:

.. code-block:: bash

    mkdir -pv ~/.config/systemd/user
    cd ~/.config/systemd/user
    podman generate systemd --files --name saferwall-webapis-pod
    systemctl --user enable pod-saferwall-webapis-pod.service container-saferwall-webapis-pod-srv01.service

Create Admin user
-----------------

Execute the following command:

.. code-block:: bash

    curl --location --request POST 'http://127.0.0.1:8000/v1/users' --header 'Content-Type: application/json' --data-raw '{
        "username": "admin",
        "email": "admin@example.com",
        "password": "abcde12345"
    }'

Go to Couchbase and edit the admin user. Make sure to set the following:

* "confirmed": true
* "admin": true

Deploy Saferwall UI
-------------------

From the project root directory, ``cd`` into ``deployment/production/ui``:

.. code-block:: bash

    cd deployment/production/ui

Create config files:

.. code-block:: bash

    cp -v configmaps/saferwall-ui.yaml{.example,}

Create pod file:

.. code-block:: bash

    cp -v saferwall-ui-pod.yaml{.example,}

For SELinux platform, label the following files to allow to be mounted into container:

.. code-block:: bash

    chcon -R -v -t container_file_t ./configs

Load SELinux security policy:

.. code-block:: bash

    sudo semodule -i selinux/saferwall_ui.cil /usr/share/udica/templates/base_container.cil

Verify that the SELinux module exists:

.. code-block:: bash

    sudo semodule --list | grep -e "saferwall_ui"

Deploy Saferwall UI:

.. code-block:: bash

    podman play kube --configmap configmaps/saferwall-ui.yaml --seccomp-profile-root ./seccomp saferwall-ui-pod.yaml

Create systemd files to run at startup:

.. code-block:: bash

    mkdir -pv ~/.config/systemd/user
    cd ~/.config/systemd/user
    podman generate systemd --files --name saferwall-ui-pod
    systemctl --user enable pod-saferwall-ui-pod.service container-saferwall-ui-pod-srv01.service

Deploy Saferwall Orchestrator
-----------------------------

From the project root directory, ``cd`` into ``deployment/production/orchestrator``:

.. code-block:: bash

    cd deployment/production/orchestrator

Create config files and fix permissions:

.. code-block:: bash

    cp -v configmaps/saferwall-orchestrator.yaml{.example,}
    cp -v configs/local.toml{.example,}
    chmod og+r configs/local.toml

Create pod file:

.. code-block:: bash

    cp -v saferwall-orchestrator-pod.yaml{.example,}

For SELinux platform, label the following files to allow to be mounted into container:

.. code-block:: bash

    chcon -R -v -t container_file_t ./configs

Load SELinux security policy:

.. code-block:: bash

    sudo semodule -i selinux/saferwall_orchestrator.cil /usr/share/udica/templates/base_container.cil

Verify that the SELinux module exists:

.. code-block:: bash

    sudo semodule --list | grep -e "saferwall_orchestrator"

Deploy Saferwall Orchestrator:

.. code-block:: bash

    podman play kube --configmap configmaps/saferwall-orchestrator.yaml --seccomp-profile-root ./seccomp saferwall-orchestrator-pod.yaml

Create systemd files to run at startup:

.. code-block:: bash

    mkdir -pv ~/.config/systemd/user
    cd ~/.config/systemd/user
    podman generate systemd --files --name saferwall-orchestrator-pod
    systemctl --user enable pod-saferwall-orchestrator-pod.service container-saferwall-orchestrator-pod-srv01.service

Deploy Saferwall Aggregator
---------------------------

From the project root directory, ``cd`` into ``deployment/production/aggregator``:

.. code-block:: bash

    cd deployment/production/aggregator

Create config files and fix permissions:

.. code-block:: bash

    cp -v configmaps/saferwall-aggregator.yaml{.example,}
    cp -v configs/local.toml{.example,}
    chmod og+r configs/local.toml

Create pod file:

.. code-block:: bash

    cp -v saferwall-aggregator-pod.yaml{.example,}

For SELinux platform, label the following files to allow to be mounted into container:

.. code-block:: bash

    chcon -R -v -t container_file_t ./configs

Load SELinux security policy:

.. code-block:: bash

    sudo semodule -i selinux/saferwall_aggregator.cil /usr/share/udica/templates/base_container.cil

Verify that the SELinux module exists:

.. code-block:: bash

    sudo semodule --list | grep -e "saferwall_aggregator"

Deploy Saferwall Aggregator:

.. code-block:: bash

    podman play kube --configmap configmaps/saferwall-aggregator.yaml --seccomp-profile-root ./seccomp saferwall-aggregator-pod.yaml

Create systemd files to run at startup:

.. code-block:: bash

    mkdir -pv ~/.config/systemd/user
    cd ~/.config/systemd/user
    podman generate systemd --files --name saferwall-aggregator-pod
    systemctl --user enable pod-saferwall-aggregator-pod.service container-saferwall-aggregator-pod-srv01.service

Deploy Saferwall Postprocessor
------------------------------

From the project root directory, ``cd`` into ``deployment/production/postprocessor``:

.. code-block:: bash

    cd deployment/production/postprocessor

Create config files and fix permissions:

.. code-block:: bash

    cp -v configmaps/saferwall-postprocessor.yaml{.example,}
    cp -v configs/local.toml{.example,}
    chmod og+r configs/local.toml

Create pod file:

.. code-block:: bash

    cp -v saferwall-postprocessor-pod.yaml{.example,}

For SELinux platform, label the following files to allow to be mounted into container:

.. code-block:: bash

    chcon -R -v -t container_file_t ./configs

Load SELinux security policy:

.. code-block:: bash

    sudo semodule -i selinux/saferwall_postprocessor.cil /usr/share/udica/templates/base_container.cil

Verify that the SELinux module exists:

.. code-block:: bash

    sudo semodule --list | grep -e "saferwall_postprocessor"

Deploy Saferwall Postprocessor:

.. code-block:: bash

    podman play kube --configmap configmaps/saferwall-postprocessor.yaml --seccomp-profile-root ./seccomp saferwall-postprocessor-pod.yaml

Create systemd files to run at startup:

.. code-block:: bash

    mkdir -pv ~/.config/systemd/user
    cd ~/.config/systemd/user
    podman generate systemd --files --name saferwall-postprocessor-pod
    systemctl --user enable pod-saferwall-postprocessor-pod.service container-saferwall-postprocessor-pod-srv01.service

Deploy Saferwall Pe
-------------------

From the project root directory, ``cd`` into ``deployment/production/pe``:

.. code-block:: bash

    cd deployment/production/pe

Create config files and fix permissions:

.. code-block:: bash

    cp -v configmaps/saferwall-pe.yaml{.example,}
    cp -v configs/local.toml{.example,}
    chmod og+r configs/local.toml

Create pod file:

.. code-block:: bash

    cp -v saferwall-pe-pod.yaml{.example,}

For SELinux platform, label the following files to allow to be mounted into container:

.. code-block:: bash

    chcon -R -v -t container_file_t ./configs

Load SELinux security policy:

.. code-block:: bash

    sudo semodule -i selinux/saferwall_pe.cil /usr/share/udica/templates/base_container.cil

Verify that the SELinux module exists:

.. code-block:: bash

    sudo semodule --list | grep -e "saferwall_pe"

Deploy Saferwall PE:

.. code-block:: bash

    podman play kube --configmap configmaps/saferwall-pe.yaml --seccomp-profile-root ./seccomp saferwall-pe-pod.yaml

Create systemd files to run at startup:

.. code-block:: bash

    mkdir -pv ~/.config/systemd/user
    cd ~/.config/systemd/user
    podman generate systemd --files --name saferwall-pe-pod
    systemctl --user enable pod-saferwall-pe-pod.service container-saferwall-pe-pod-srv01.service

Deploy Saferwall Meta
---------------------

From the project root directory, ``cd`` into ``deployment/production/meta``:

.. code-block:: bash

    cd deployment/production/meta

Create config files and fix permissions:

.. code-block:: bash

    cp -v configmaps/saferwall-meta.yaml{.example,}
    cp -v configs/local.toml{.example,}
    chmod og+r configs/local.toml

Create pod file:

.. code-block:: bash

    cp -v saferwall-meta-pod.yaml{.example,}

For SELinux platform, label the following files to allow to be mounted into container:

.. code-block:: bash

    chcon -R -v -t container_file_t ./configs

Load SELinux security policy:

.. code-block:: bash

    sudo semodule -i selinux/saferwall_meta.cil /usr/share/udica/templates/base_container.cil

Verify that the SELinux module exists:

.. code-block:: bash

    sudo semodule --list | grep -e "saferwall_pe"

Deploy Saferwall Meta:

.. code-block:: bash

    podman play kube --configmap configmaps/saferwall-meta.yaml --seccomp-profile-root ./seccomp saferwall-meta-pod.yaml

Create systemd files to run at startup:

.. code-block:: bash

    mkdir -pv ~/.config/systemd/user
    cd ~/.config/systemd/user
    podman generate systemd --files --name saferwall-meta-pod
    systemctl --user enable pod-saferwall-meta-pod.service container-saferwall-meta-pod-srv01.service

Deploy Saferwall AV ClamAV
--------------------------

From the project root directory, ``cd`` into ``deployment/production/av-clamav``:

.. code-block:: bash

    cd deployment/production/av-clamav

Create config files:

.. code-block:: bash

    cp -v configmaps/saferwall-av-clamav.yaml{.example,}
    cp -v configs/local.toml{.example,}
    chmod og+r configs/local.toml

Create pod file:

.. code-block:: bash

    cp -v saferwall-av-clamav-pod.yaml{.example,}

For SELinux platform, label the following files to allow to be mounted into container:

.. code-block:: bash

    chcon -R -v -t container_file_t ./configs

Load SELinux security policy:

.. code-block:: bash

    sudo semodule -i selinux/saferwall_av_clamav.cil /usr/share/udica/templates/base_container.cil

Verify that the SELinux module exists:

.. code-block:: bash

    sudo semodule --list | grep -e "saferwall_av_clamav"

Deploy:

.. code-block:: bash

    podman play kube --configmap configmaps/saferwall-av-clamav.yaml --seccomp-profile-root ./seccomp saferwall-av-clamav-pod.yaml

Create systemd files to run at startup:

.. code-block:: bash

    mkdir -pv ~/.config/systemd/user
    cd ~/.config/systemd/user
    podman generate systemd --files --name saferwall-av-clamav-pod
    systemctl --user enable pod-saferwall-av-clamav-pod.service container-saferwall-av-clamav-pod-srv01.service

Deploy Saferwall AV Comodo
--------------------------

From the project root directory, ``cd`` into ``deployment/production/av-comodo``:

.. code-block:: bash

    cd deployment/production/av-comodo

Create config files and fix permissions:

.. code-block:: bash

    cp -v configmaps/saferwall-av-comodo.yaml{.example,}
    cp -v configs/local.toml{.example,}
    chmod og+r configs/local.toml

Create pod file:

.. code-block:: bash

    cp -v saferwall-av-comodo-pod.yaml{.example,}

For SELinux platform, label the following files to allow to be mounted into container:

.. code-block:: bash

    chcon -R -v -t container_file_t ./configs

Load SELinux security policy:

.. code-block:: bash

    sudo semodule -i selinux/saferwall_av_comodo.cil /usr/share/udica/templates/base_container.cil

Verify that the SELinux module exists:

.. code-block:: bash

    sudo semodule --list | grep -e "saferwall_av_comodo"

Deploy:

.. code-block:: bash

    podman play kube --configmap configmaps/saferwall-av-comodo.yaml --seccomp-profile-root ./seccomp saferwall-av-comodo-pod.yaml

Create systemd files to run at startup:

.. code-block:: bash

    mkdir -pv ~/.config/systemd/user
    cd ~/.config/systemd/user
    podman generate systemd --files --name saferwall-av-comodo-pod
    systemctl --user enable pod-saferwall-av-comodo-pod.service container-saferwall-av-comodo-pod-srv01.service

Deploy Saferwall AV Avira
-------------------------

From the project root directory, ``cd`` into ``deployment/production/av-avira``:

.. code-block:: bash

    cd deployment/production/av-avira

Create config files and fix permissions:

.. code-block:: bash

    cp -v configmaps/saferwall-av-avira.yaml{.example,}
    cp -v configs/local.toml{.example,}
    chmod og+r configs/local.toml

Create pod file:

.. code-block:: bash

    cp -v saferwall-av-avira-pod.yaml{.example,}

For SELinux platform, label the following files to allow to be mounted into container:

.. code-block:: bash

    chcon -R -v -t container_file_t ./configs

Load SELinux security policy:

.. code-block:: bash

    sudo semodule -i selinux/saferwall_av_avira.cil /usr/share/udica/templates/base_container.cil

Verify that the SELinux module exists:

.. code-block:: bash

    sudo semodule --list | grep -e "saferwall_av_avira"

Deploy:

.. code-block:: bash

    podman play kube --configmap configmaps/saferwall-av-avira.yaml --seccomp-profile-root ./seccomp saferwall-av-avira-pod.yaml

Create systemd files to run at startup:

.. code-block:: bash

    mkdir -pv ~/.config/systemd/user
    cd ~/.config/systemd/user
    podman generate systemd --files --name saferwall-av-avira-pod
    systemctl --user enable pod-saferwall-av-avira-pod.service container-saferwall-av-avira-pod-srv01.service
