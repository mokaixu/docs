It is strongly recommended that you regularly upgrade your CockroachDB version in order to pick up bug fixes, performance improvements, and new features. The [CockroachDB upgrade documentation](upgrade-cockroach-version.html) describes how to perform a "rolling upgrade" of a CockroachDB cluster by stopping and restarting nodes one at a time. This is to ensure that the cluster remains available during the upgrade.

The corresponding process on Kubernetes is a [staged update](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/#staging-an-update), in which the Docker image is updated in the CockroachDB StatefulSet and then applied to the pods one at a time.

1. Decide how the upgrade will be finalized.

    {{site.data.alerts.callout_info}}
    This step is relevant only when upgrading from v19.2.x to v20.1. For upgrades within the v20.1.x series, skip this step.
    {{site.data.alerts.end}}

    By default, after all nodes are running the new version, the upgrade process will be **auto-finalized**. This will enable certain performance improvements and bug fixes introduced in v20.1. After finalization, however, it will no longer be possible to perform a downgrade to v19.2. In the event of a catastrophic failure or corruption, the only option will be to start a new cluster using the old binary and then restore from one of the backups created prior to performing the upgrade.

    We recommend disabling auto-finalization so you can monitor the stability and performance of the upgraded cluster before finalizing the upgrade:

    {% if page.secure == true %}

    1. Get a shell into the pod with the `cockroach` binary created earlier and start the CockroachDB [built-in SQL client](cockroach-sql.html):

        <section class="filter-content" markdown="1" data-scope="manual">
        {% include copy-clipboard.html %}
        ~~~ shell
        $ kubectl exec -it cockroachdb-client-secure \-- ./cockroach sql \
        --certs-dir=/cockroach-certs \
        --host=cockroachdb-public
        ~~~
        </section>

        <section class="filter-content" markdown="1" data-scope="helm">
        {% include copy-clipboard.html %}
        ~~~ shell
        $ kubectl exec -it cockroachdb-client-secure \
        -- ./cockroach sql \
        --certs-dir=/cockroach-certs \
        --host=my-release-cockroachdb-public
        ~~~
        </section>

    {% else %}

    1. Launch a temporary interactive pod and start the [built-in SQL client](cockroach-sql.html) inside it:

        <section class="filter-content" markdown="1" data-scope="manual">
        {% include copy-clipboard.html %}
        ~~~ shell
        $ kubectl run cockroachdb -it \
        --image=cockroachdb/cockroach \
        --rm \
        --restart=Never \
        -- sql \
        --insecure \
        --host=cockroachdb-public
        ~~~
        </section>

        <section class="filter-content" markdown="1" data-scope="helm">
        {% include copy-clipboard.html %}
        ~~~ shell
        $ kubectl run cockroachdb -it \
        --image=cockroachdb/cockroach \
        --rm \
        --restart=Never \
        -- sql \
        --insecure \
        --host=my-release-cockroachdb-public
        ~~~
        </section>

    {% endif %}

    1. Set the `cluster.preserve_downgrade_option` [cluster setting](cluster-settings.html) to the version you are upgrading from:

        {% include copy-clipboard.html %}
        ~~~ sql
        > SET CLUSTER SETTING cluster.preserve_downgrade_option = '19.2';
        ~~~

    1. Exit the SQL shell and delete the temporary pod:

        {% include copy-clipboard.html %}
        ~~~ sql
        > \q
        ~~~

1. Add a [partition](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/#staging-an-update) to the update strategy defined in the StatefulSet. Only the pods numbered greater than or equal to the partition value will be updated. For a cluster with 3 pods (e.g., `cockroachdb-0`, `cockroachdb-1`, `cockroachdb-2`) the partition value should be 2:

    <section class="filter-content" markdown="1" data-scope="manual">
    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl patch statefulset cockroachdb \
    -p='{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":2}}}}'
    ~~~

    ~~~
    statefulset.apps/cockroachdb patched
    ~~~
    </section>

    <section class="filter-content" markdown="1" data-scope="helm">
    {% include copy-clipboard.html %}
    ~~~ shell
    $ helm upgrade \
    my-release \
    cockroachdb/cockroachdb \
    --set statefulset.updateStrategy.rollingUpdate.partition=2
    ~~~
    </section>

1. Kick off the upgrade process by changing the Docker image used in the CockroachDB StatefulSet:

    <section class="filter-content" markdown="1" data-scope="manual">
    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl patch statefulset cockroachdb \
    --type='json' \
    -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"cockroachdb/cockroach:{{page.release_info.version}}"}]'
    ~~~

    ~~~
    statefulset.apps/cockroachdb patched
    ~~~
    </section>

    <section class="filter-content" markdown="1" data-scope="helm">

    {{site.data.alerts.callout_info}}
    For Helm, you must remove the cluster initialization job from when the cluster was created before the cluster version can be changed.
    {{site.data.alerts.end}}

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl delete job my-release-cockroachdb-init
    ~~~

    {% include copy-clipboard.html %}
    ~~~ shell
    $ helm upgrade \
    my-release \
    cockroachdb/cockroachdb \
    --set image.tag={{page.release_info.version}} \
    --reuse-values
    ~~~
    </section>

1. Check the status of your cluster's pods. You should see one of them being restarted:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get pods
    ~~~

    <section class="filter-content" markdown="1" data-scope="manual">
    ~~~
    NAME            READY     STATUS        RESTARTS   AGE
    cockroachdb-0   1/1       Running       0          2m
    cockroachdb-1   1/1       Running       0          2m
    cockroachdb-2   0/1       Terminating   0          1m
    ...
    ~~~
    </section>

    <section class="filter-content" markdown="1" data-scope="helm">
    ~~~
    NAME                                READY     STATUS              RESTARTS   AGE
    my-release-cockroachdb-0            1/1       Running             0          2m
    my-release-cockroachdb-1            1/1       Running             0          3m
    my-release-cockroachdb-2            0/1       ContainerCreating   0          25s
    my-release-cockroachdb-init-nwjkh   0/1       ContainerCreating   0          6s
    ...
    ~~~

    {{site.data.alerts.callout_info}}
    Ignore the pod for cluster initialization. It is re-created as a byproduct of the StatefulSet configuration but does not impact your existing cluster.
    {{site.data.alerts.end}}
    </section>    

1. After the pod has been restarted with the new image, get a shell into the pod and start the CockroachDB [built-in SQL client](cockroach-sql.html):

    {% if page.secure == true %}
    <section class="filter-content" markdown="1" data-scope="manual">
    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl exec -it cockroachdb-client-secure \-- ./cockroach sql \
    --certs-dir=/cockroach-certs \
    --host=cockroachdb-public
    ~~~
    </section>

    <section class="filter-content" markdown="1" data-scope="helm">
    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl exec -it cockroachdb-client-secure \
    -- ./cockroach sql \
    --certs-dir=/cockroach-certs \
    --host=my-release-cockroachdb-public
    ~~~
    </section>

    {% else %}

    <section class="filter-content" markdown="1" data-scope="manual">
    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl run cockroachdb -it \
    --image=cockroachdb/cockroach \
    --rm \
    --restart=Never \
    -- sql \
    --insecure \
    --host=cockroachdb-public
    ~~~
    </section>

    <section class="filter-content" markdown="1" data-scope="helm">
    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl run cockroachdb -it \
    --image=cockroachdb/cockroach \
    --rm \
    --restart=Never \
    -- sql \
    --insecure \
    --host=my-release-cockroachdb-public
    ~~~
    </section>    
    {% endif %}

1. Run the following SQL query to verify that the number of underreplicated ranges is zero:

    {% include copy-clipboard.html %}
    ~~~ sql
    SELECT sum((metrics->>'ranges.underreplicated')::DECIMAL)::INT AS ranges_underreplicated FROM crdb_internal.kv_store_status;
    ~~~

    ~~~
      ranges_underreplicated
    --------------------------
                           0
    (1 row)        
    ~~~

    This indicates that it is safe to proceed to the next pod.

1. Exit the SQL shell:

    {% include copy-clipboard.html %}
    ~~~ sql
    > \q
    ~~~

1. Decrement the partition value by 1 to allow the next pod in the cluster to update:

    <section class="filter-content" markdown="1" data-scope="manual">
    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl patch statefulset cockroachdb \
    -p='{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":1}}}}'
    ~~~

    ~~~
    statefulset.apps/cockroachdb patched
    ~~~
    </section>

    <section class="filter-content" markdown="1" data-scope="helm">
    {% include copy-clipboard.html %}
    ~~~ shell
    $ helm upgrade \
    my-release \
    cockroachdb/cockroachdb \
    --set statefulset.updateStrategy.rollingUpdate.partition=1 \
    ~~~
    </section>

1. Repeat steps 4-8 until all pods have been restarted and are running the new image (the final partition value should be `0`).

1. Check the image of each pod to confirm that all have been upgraded:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ kubectl get pods \
    -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}'
    ~~~

    <section class="filter-content" markdown="1" data-scope="manual">
    ~~~
    cockroachdb-0   cockroachdb/cockroach:{{page.release_info.version}}
    cockroachdb-1   cockroachdb/cockroach:{{page.release_info.version}}
    cockroachdb-2   cockroachdb/cockroach:{{page.release_info.version}}
    ...
    ~~~
    </section>

    <section class="filter-content" markdown="1" data-scope="helm">
    ~~~
    my-release-cockroachdb-0    cockroachdb/cockroach:{{page.release_info.version}}
    my-release-cockroachdb-1    cockroachdb/cockroach:{{page.release_info.version}}
    my-release-cockroachdb-2    cockroachdb/cockroach:{{page.release_info.version}}
    ...
    ~~~
    </section>

    You can also check the CockroachDB version of each node in the [Admin UI](admin-ui-cluster-overview-page.html#node-details).


1. Finish the upgrade.

    {{site.data.alerts.callout_info}}
    This step is relevant only when upgrading from v19.2.x to v20.1. For upgrades within the v20.1.x series, skip this step.
    {{site.data.alerts.end}}

    If you disabled auto-finalization in step 1 above, monitor the stability and performance of your cluster for as long as you require to feel comfortable with the upgrade (generally at least a day). If during this time you decide to roll back the upgrade, repeat the rolling restart procedure with the old binary.

    Once you are satisfied with the new version, re-enable auto-finalization:

    {% if page.secure == true %}

    1. Get a shell into the pod with the `cockroach` binary created earlier and start the CockroachDB [built-in SQL client](cockroach-sql.html):

        <section class="filter-content" markdown="1" data-scope="manual">
        {% include copy-clipboard.html %}
        ~~~ shell
        $ kubectl exec -it cockroachdb-client-secure \
        -- ./cockroach sql \
        --certs-dir=/cockroach-certs \
        --host=cockroachdb-public
        ~~~
        </section>

        <section class="filter-content" markdown="1" data-scope="helm">
        {% include copy-clipboard.html %}
        ~~~ shell
        $ kubectl exec -it cockroachdb-client-secure \
        -- ./cockroach sql \
        --certs-dir=/cockroach-certs \
        --host=my-release-cockroachdb-public
        ~~~
        </section>

    {% else %}

    1. Launch a temporary interactive pod and start the [built-in SQL client](cockroach-sql.html) inside it:

        <section class="filter-content" markdown="1" data-scope="manual">
        {% include copy-clipboard.html %}
        ~~~ shell
        $ kubectl run cockroachdb -it \
        --image=cockroachdb/cockroach \
        --rm \
        --restart=Never \
        -- sql \
        --insecure \
        --host=cockroachdb-public
        ~~~
        </section>

        <section class="filter-content" markdown="1" data-scope="helm">
        {% include copy-clipboard.html %}
        ~~~ shell
        $ kubectl run cockroachdb -it \
        --image=cockroachdb/cockroach \
        --rm \
        --restart=Never \
        -- sql \
        --insecure \
        --host=my-release-cockroachdb-public
        ~~~
        </section>      

    {% endif %}

    2. Re-enable auto-finalization:

        {% include copy-clipboard.html %}
        ~~~ sql
        > RESET CLUSTER SETTING cluster.preserve_downgrade_option;
        ~~~

    3. Exit the SQL shell and delete the temporary pod:

        {% include copy-clipboard.html %}
        ~~~ sql
        > \q
        ~~~
