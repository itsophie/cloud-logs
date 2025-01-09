---

copyright:
  years:  2024
lastupdated: "2024-12-12"

keywords:

subcollection: cloud-logs

---

{{site.data.keyword.attribute-definition-list}}


# Supporting multiline logs in the {{site.data.keyword.agent}}
{: #agent-multiline}

To support the ingestion of multiline logs from applications, like Java or Python, where errors and stack traces can span several lines, and each line is sent as a separate log entry, you must make changes to the {{site.data.keyword.agent}} configuration. The change includes the parsing required to group log lines that are supposed to be together as a single log record.
{: shortdesc}


## About multiline
{: #agent-multiline-about}

In OpenShift and Kubernetes clusters, containers can write to standard output (`stdout`) and standard error (`stderr`) streams, and the logs follow the CRI logging format.

The CRI logging format uses tags to define if a log line is a single log line or a multiline log entry. Valid values for the tag are:
- Partial (P): This tag is included in log lines that are the result of spliting a single log line into multiple lines by the runtime and the log entry has not ended yet.
- Full (F): This tag is used to indicate that the log entry is completed. It is used for a single log line entry or to indicate that it is the last line of the multiple-line entry.

By default, the {{site.data.keyword.agent}} includes the configuration of the `Tail plugin` to support multiline logs into a single log line for container runtimes that write logs following the CRI logging format into stdout and stderr.

You might also have applications, like Java or Python, where errors and stack traces can span several lines, and each line is sent as a separate log entry. These applications can generate multiple log lines that can be associated with one another into a single log line. To handle these multiline logs through the {{site.data.keyword.agent}}, you must configure the `Multiline parser`.



## Default multiline configuration
{: #agent-multiline-default}

The default multiline configuration is configured and enabled when you deploy the {{site.data.keyword.agent}}.
{: note}

In Fluent Bit, you can configure the `Multiline parser` by using the built-in multiline parser or by using a custom multiline parser.

By default, the `Tail plugin` that is configured with the {{site.data.keyword.agent}} is set with the built-in multiline `cri` parser. This parser processes logs that are generated by the CRI-O container engine and supports concatenation of log entries.

For example, the {{site.data.keyword.agent}} configuration looks as follows with the default configuration for multiline support:

```yml
    [INPUT]
        Name              tail
        Tag               kube.*
        .....
        Buffer_Chunk_Size 32KB
        Buffer_Max_Size   256KB
        Multiline.parser  cri
        Skip_Long_Lines   On
        Refresh_Interval  10
        storage.type      filesystem
        storage.pause_on_chunks_overlimit on
```
{: codeblock}


## Changing the default multiline configuration
{: #agent-multiline-parser-change}

Fluent bit supports different built-in multiline parsers that solve specific multiline parser cases such as `docker`, `cri` `java`, `go`, and `python`. For more information, see [Built-in multiline parsers](https://docs.fluentbit.io/manual/administration/configuring-fluent-bit/multiline-parsing#built-in-multiline-parsers){: external}.

To change the built-in multiline `cri` parser set by default in the {{site.data.keyword.agent}}, complete the following steps:

1. Log in to the cluster. For more information, see [Access your cluster](/docs/containers?topic=containers-access_cluster).

2. In the {{site.data.keyword.agent}} configmap, change the multiline filter provided with the default configuration from `cri` to `docker`, `java`, `go`, or `python`.

    ```yml
    [INPUT]
        Name              tail
        Tag               kube.*
        .....
        Buffer_Chunk_Size 32KB
        Buffer_Max_Size   256KB
        Multiline.parser  ENTER_VALID_VALUE           # Valid values: cri, docker, java, go, or python
        Skip_Long_Lines   On
        Refresh_Interval  10
        storage.type      filesystem
        storage.pause_on_chunks_overlimit on
    ```
    {: codeblock}

3. Restart the agent pods.

    For Kubernetes clusters, run:

    ```
    kubectl -n ibm-observe rollout restart ds/logs-agent
    ```
    {: codeblock}

    For OpenShift clusters, run:

    ```
    oc -n ibm-observe rollout restart ds/logs-agent
    ```
    {: codeblock}



## Configuring the Multiline parser
{: #agent-multiline-config-parser}

If you have applications, like Java or Python, where errors and stack traces can span several lines, and each line is sent as a separate log entry, you must configure the Multiline parser in the {{site.data.keyword.agent}}.
{: note}

Choose one of the following options to configure the {{site.data.keyword.agent}} with the Multiline parser:
- Deploy the {{site.data.keyword.agent}} by adding a new value `enableMultiline` in the `logs-values.yaml` file that you use to deploy the agent by using a Helm chart. For more information, see [Configuring the Helm chart values file for the {{site.data.keyword.agent}}](/docs/cloud-logs?topic=cloud-logs-agent-helm-os-deploy#agent-helm-os-deploy-step2) or [Configuring the Helm chart values file for the {{site.data.keyword.agent}}](/docs/cloud-logs?topic=cloud-logs-agent-helm-kube-deploy#agent-helm-kube-deploy-step2).

    A pre-defined configuration for the `MULTILINE PARSER` to support multiline logs is included starting with the {{site.data.keyword.agent}} version 1.4.1 deployment files. You must enable the `MULTILINE PARSER`.

- Update the {{site.data.keyword.agent}} to version 1.4.1 or above. You must update the Helm chart `logs-values.yaml` file with the agent version and add the value `enableMultiline` to enable multiline support. For more information, see [Update the Helm chart values file for the Logging agent](/docs/cloud-logs?topic=cloud-logs-agent-helm-update#agent-helm-update-step1).

- Modify the {{site.data.keyword.agent}} configuration with the Multiline parser. For more information, see [Modifying the {{site.data.keyword.agent}} configuration with the Multiline parser](#agent-multiline-parser-for-apps).


After you configure the Multiline parser, the default multiline {{site.data.keyword.agent}} configuration will also include a `FILTER` after the `INPUT` plug-in and a Multiline parser. The filter will apply the pattern of the configured `MULTILINE_PARSER`. The {{site.data.keyword.agent}} configuration must `@INCLUDE` the multiline filter right after the input plug-in.
{: important}

For example, after you configure the Multiline parser, the {{site.data.keyword.agent}} configmap looks as follows:

```yaml
    ......
    [INPUT]
        Name              tail
        Tag               kube.*
        .....
        Buffer_Chunk_Size 32KB
        Buffer_Max_Size   256KB
        Multiline.parser  cri
        Skip_Long_Lines   On
        Refresh_Interval  10
        storage.type      filesystem
        storage.pause_on_chunks_overlimit on

    filter-multiline.conf: |
    [FILTER]
        Name              multiline
        Match             *
        Multiline.parser  multiline_pattern
        Multiline.key_content log

    ......
    [MULTILINE_PARSER]
        Name            multiline_pattern
        Type            regex
        Flush_timeout   1000
        Rule            "start_state"     "/^.*:[\s]*$/"               "colon_cont"
        Rule            "start_state"     "/^.*$/"                     "cont"
        Rule            "cont"            "/^[\s]+.*:[\s]*$/"          "colon_cont"
        Rule            "cont"            "/^[\s]+.*$/"                "cont"
        Rule            "cont"            "/^}$/"                      "cont"
        Rule            "cont"            "/^\s*$/"                    "cont"
        Rule            "colon_cont"      "/^.*:[\s]*$/"               "colon_cont"
        Rule            "colon_cont"      "/^.*$/"                     "cont"
    ......
```
{: codeblock}


The pre-defined configuration assumes that any line ending with a colon (`:`) is a multiline.
{: note}


## Adding multiline support for apps
{: #agent-multiline-parser-for-apps}

If you have the {{site.data.keyword.agent}} deployed and you have apps like Java or Python, where errors and stack traces can span several lines, and each line is sent as a separate log entry, you can update your agent configuration manually and configure the Multiline parser.
{: note}

Complete the following steps to add multiline support in the {{site.data.keyword.agent}} for apps like Java or Python, where errors and stack traces can span several lines, and each line is sent as a separate log entry:

1. Log in to the cluster. For more information, see [Access your cluster](/docs/containers?topic=containers-access_cluster).

2. In the {{site.data.keyword.agent}} configmap, add the Multiline parser.

    The {{site.data.keyword.agent}} configuration must `@INCLUDE` the multiline filter right after the input plug-in.{: important}

    ```yaml
    ......
    [INPUT]
        Name              tail
        Tag               kube.*
        .....
        Buffer_Chunk_Size 32KB
        Buffer_Max_Size   256KB
        Multiline.parser  cri
        Skip_Long_Lines   On
        Refresh_Interval  10
        storage.type      filesystem
        storage.pause_on_chunks_overlimit on

    filter-multiline.conf: |
    [FILTER]
        Name              multiline
        Match             *
        Multiline.parser  multiline_pattern
        Multiline.key_content log

    ......
    [MULTILINE_PARSER]
        Name            multiline_pattern
        Type            regex
        Flush_timeout   1000
        Rule            "start_state"     "/^.*:[\s]*$/"               "colon_cont"
        Rule            "start_state"     "/^.*$/"                     "cont"
        Rule            "cont"            "/^[\s]+.*:[\s]*$/"          "colon_cont"
        Rule            "cont"            "/^[\s]+.*$/"                "cont"
        Rule            "cont"            "/^}$/"                      "cont"
        Rule            "cont"            "/^\s*$/"                    "cont"
        Rule            "colon_cont"      "/^.*:[\s]*$/"               "colon_cont"
        Rule            "colon_cont"      "/^.*$/"                     "cont"
    ......
    ```
    {: codeblock}

3. Restart the agent pods.

    For Kubernetes clusters, run:

    ```
    kubectl -n ibm-observe rollout restart ds/logs-agent
    ```
    {: codeblock}

    For OpenShift clusters, run:

    ```
    oc -n ibm-observe rollout restart ds/logs-agent
    ```
    {: codeblock}


The pre-defined configuration assumes that any line ending with a colon (`:`) is a multiline.
{: note}



## Changing the multiline configuration to handle a colon at the end of a line
{: #agent-multiline-modify-configuration}

The default configuration assumes that any line ending with a colon (`:`) is a multiline. If this is not the case in your environment, you will need to change the parser to ignore the final `:` by removing `colon_cont` in the parser. With this change the multiline parser should look like the following:

Complete the following steps:

1. Log in to the cluster. For more information, see [Access your cluster](/docs/containers?topic=containers-access_cluster).

2. In the {{site.data.keyword.agent}} configmap, add the Multiline parser.

    The {{site.data.keyword.agent}} configuration must `@INCLUDE` the multiline filter right after the input plug-in.{: important}

    ```yaml
    ......
    [INPUT]
        Name              tail
        Tag               kube.*
        .....
        Buffer_Chunk_Size 32KB
        Buffer_Max_Size   256KB
        Multiline.parser  cri
        Skip_Long_Lines   On
        Refresh_Interval  10
        storage.type      filesystem
        storage.pause_on_chunks_overlimit on

    filter-multiline.conf: |
    [FILTER]
        Name              multiline
        Match             *
        Multiline.parser  multiline_pattern
        Multiline.key_content log

    ......
    [MULTILINE_PARSER]
        Name            multiline_pattern
        Type            regex
        Flush_timeout   1000
        Rule            "start_state"     "/^.*$/"                     "cont"
        Rule            "cont"            "/^[\s]+.*$/"                "cont"
        Rule            "cont"            "/^}$/"                      "cont"
        Rule            "cont"            "/^\s*$/"                    "cont"
    ```
    {: codeblock}

3. Restart the agent pods.

    For Kubernetes clusters, run:

    ```
    kubectl -n ibm-observe rollout restart ds/logs-agent
    ```
    {: codeblock}

    For OpenShift clusters, run:

    ```
    oc -n ibm-observe rollout restart ds/logs-agent
    ```
    {: codeblock}


The {{site.data.keyword.agent}} configuration must also include a `FILTER` after the `INPUT` plug-in. The filter will apply the pattern of the configured `MULTILINE_PARSER`. The `Name` value of the `MULTILINE_PARSER` must match the `Multiline.parser` value in the `FILTER`.


## Adding a custom multiline parser
{: #agent-multiline-new-parser}

To create a custom multiline parser for use with the {{site.data.keyword.agent}}, follow the instructions in [Configurable Multiline Parsers](https://docs.fluentbit.io/manual/administration/configuring-fluent-bit/multiline-parsing#configurable-multiline-parsers).{: external} You will define a custom regex to determine the multiline pattern.

The {{site.data.keyword.agent}} configuration must also include a `FILTER` after the `INPUT` plug-in. The filter will apply the pattern of the configured `MULTILINE_PARSER`. The `Name` value of the `MULTILINE_PARSER` must match the `Multiline.parser` value in the `FILTER`.

```yaml
filter-multiline.conf: |
    [FILTER]
        Name              multiline
        Match             *
        Multiline.parser  INSERT_CUSTOM_PARSER_NAME
        Multiline.key_content log
```
{: codeblock}