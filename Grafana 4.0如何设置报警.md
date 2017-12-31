## Grafana报警配置  

Grafana从版本4.0开始支持报警功能，可配置的报警方式有很多种，如：邮件报警，即时通信工具报警(钉钉、Slack、Line)，Webhook报警等十几种方式。下面主要介绍下常用的邮件与短信报警方式的配置。  

### 报警通道配置  

如图所示  

![Alert Notification Menu](http://docs.grafana.org/img/docs/v43/alert_notifications_menu.png)   

进入`Notification channels`，点击`+New Channel`即可创建报警方式。

#### 邮件报警  

**如图所示**  

![Email Channel](https://github.com/coolzhang/myblog/blob/master/misc/grafana_alert_email.png)  

**配置说明**  

* `Type`定义报警通道类型，选择`Email`类型即可支持邮件报警。  
* `Email addresses`下面填写需要接受报警的邮件地址，以逗号分隔。  
* `Include image`勾选后，邮件内容会加载报警时刻的监控图。  

#### 短信报警  

**如图所示**  

![SMS Channel](https://github.com/coolzhang/myblog/blob/master/misc/grafana_alert_webhook.png)  

**配置说明**  

* `Type`定义报警通道类型，选择`webhook`类型即可支持短信报警。  
* `Webhook settings` - `Url`指定报警触发后调用的web服务地址，Grafana会将报警详细内容以如下JSON格式POST给Url中指定的服务，收到后只需解析出所需字段作为报警内容即可（我们选择的是title，metric，value）。

```
{
  "evalMatches": [
    {
      "metric": "IO_running",
      "value": 0,
      "tags": {
        "__name__": "mysql_slave_status_slave_io_running",
        "job": "mysql",
        "master_uuid": "33801cbf-ea01-11e7-8e19-52540058d440",
        "master_host": "10.1.8.1",
        "instance": "cmug-mysqlslave-10.1.8.1-3306"
      }
    }
  ],
  "ruleId": 15,
  "title": "[Alerting] cmug-mysqlslave-10.1.8.1-3306 | Replication Alert",
  "ruleUrl": "http:\/\/10.1.8.55:3000\/graph\/dashboard\/db\/alert-mysql-replication?fullscreen&edit&tab=alert&panelId=1&orgId=1",
  "state": "alerting",
  "ruleName": "cmug-mysqlslave-10.1.8.1-3306 | Replication Alert"
}
```
* `Webhook settings` - `Http method`指定web请求的方法，是POST还是PUT。  

### 报警模板配置  

由于目前Grafana只能对具体的主机配置报警，实例名不支持变量，即instance="cmug-mysqlslave-10.1.8.1-3306"支持，instance="$host"不支持。  

如图所示  

![Graph Metrics](https://github.com/coolzhang/myblog/blob/master/misc/grafana_graph_metrics.png)  

配置说明  

* `Legend format`定义内容对应JSON串中的`metric`字段。  

通过对`Graph`-`Alert`进行配置，来设置具体主机监控项的报警规则。  

如图所示  

![Graph Alert Config](https://github.com/coolzhang/myblog/blob/master/misc/grafana_graph_alert_config.png)  

![Graph Alert Notification](https://github.com/coolzhang/myblog/blob/master/misc/grafana_graph_notification.png)  
  
配置说明  

* `Alert config` - `name`定义内容对应JSON串中的`title`字段。  
* `Notificaton` - `Send to`配置报警通道，选择之前配置好的报警方式，即：邮件报警，还是短信报警。  

### 不足之处  

目前Grafana的报警功能还比较弱，配置比较繁琐，需要对每个主机的报警项单独建立`Graph`，逐个配置报警选项。  

### 相关连接  

* [Alerting Engine & Rules Guide](http://docs.grafana.org/alerting/rules/)  
* [Alert Notifications](http://docs.grafana.org/alerting/notifications/)  




