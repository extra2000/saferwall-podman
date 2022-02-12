Updating Antivirus
==================

Comodo
------

.. code-block:: bash

    podman exec -it saferwall-av-comodo-pod-srv01 bash -c 'wget $COMODO_UPDATE -O $COMODO_BASES_CAV_PATH'
    podman exec -it -u root saferwall-av-comodo-pod-srv01 bash -c 'echo -n "$(date +%s)" > $COMODO_DB_UPDATE_DATE'
    podman container stop saferwall-av-comodo-pod-srv01
    podman container start saferwall-av-comodo-pod-srv01

Avira
-----

.. code-block:: bash

    podman exec -it saferwall-av-avira-pod-srv01 bash
    mkdir $AVIRA_TMP
    wget $AVIRA_FUSEBUNDLE -P $AVIRA_TMP
    unzip -o $AVIRA_TMP/avira_fusebundlegen-linux_glibc22-en.zip -d $AVIRA_TMP
    rm -f $AVIRA_INSTALL_DIR/*.vdf
    $AVIRA_TMP/fusebundle.bin
    unzip -o $AVIRA_TMP/install/fusebundle-linux_glibc22-int.zip -d $AVIRA_INSTALL_DIR
    rm -rf $AVIRA_TMP
    exit
    podman exec -it -u root saferwall-av-avira-pod-srv01 bash -c 'echo -n "$(date +%s)" > $AVIRA_DB_UPDATE_DATE'
    podman container stop saferwall-av-avira-pod-srv01
    podman container start saferwall-av-avira-pod-srv01
