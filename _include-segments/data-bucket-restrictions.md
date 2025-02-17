## Data bucket restrictions
{: #cos_databucket_restrictions}



{{site.data.keyword.cos_full_notm}} buckets used by {{site.data.keyword.logs_full_notm}} as data buckets can be configured only with the following [storage classes](/docs/cloud-object-storage?topic=cloud-object-storage-classes):

* Smart Tier
* Standard

The following storage classes are not supported by {{site.data.keyword.logs_full_notm}} as data buckets:

* Vault
* Cold Vault

{{site.data.keyword.cos_full_notm}} allows you to define [archive rules](/docs/cloud-object-storage?topic=cloud-object-storage-archive) on buckets that archive objects automatically after the defined time period. Archived objects have a lower cost than regular objects, but need to be restored before they can be read again.

{{site.data.keyword.logs_full_notm}} cannot read archived objects. {{site.data.keyword.logs_full_notm}} searching of archived objects in the [All Logs](/docs/cloud-logs?topic=cloud-logs-query-data#query-data-all-logs) view, or querying in [Archive queries](/docs/cloud-logs?topic=cloud-logs-query-archive-data-bucket), returns an error message.

{{site.data.keyword.cos_full_notm}} buckets used as {{site.data.keyword.logs_full_notm}} data buckets must not define archive rules that immediately archive objects, or archive objects within a few hours.

If you do not need to search logs older than a certain time period, for example, a month, you can define an {{site.data.keyword.cos_full_notm}} archive rule to archive objects older that the time period required for searching. Do not configure archiving for a period of less than 7 days.

By archiving data that you do not need to search, you can retain the log data at a reduced cost. If required, you can [restore archived objects](/docs/cloud-object-storage?topic=cloud-object-storage-archive#archive-restore-cli) if you need to search the data by using {{site.data.keyword.logs_full_notm}} in the future. 
