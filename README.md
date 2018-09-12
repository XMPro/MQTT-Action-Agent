# MQTT-Action-Agent

## Prerequisites
- The port on which MQTT is listening should be accessible on the network
- Visual Studio (any version that supports .Net Core 2.1)
- [XMPro IoT Framework NuGet package](https://www.nuget.org/packages/XMPro.IOT.Framework/3.0.2-beta)
- Please see the [Building an Agent for XMPro IoT](https://docs.xmpro.com/lessons/writing-an-agent-for-xmpro-iot/) guide for a better understanding of how the XMPro IoT Framework works

## Description
The MQTT action agent posts data to a MQTT channel to which a device might have subscribed for instructions or control commands.

## How the code works
All settings referred to in the code need to correspond with the settings defined in the template that has been created for the agent using the Stream Integration Manager. Refer to the [Stream Integration Manager](https://docs.xmpro.com/courses/packaging-an-agent-using-stream-integration-manager/) guide for instructions on how to define the settings in the template and package the agent after building the code. 

After packaging the agent, you can upload it to XMPro IoT and start using it.

### Settings
When a user needs to use the *MQTT Action Agent*, they need to provide a Broker address, and a topic. Retrieve these values from the configuration using the following code: 
```csharp
private Configuration config;
private MqttClient client;
private string Broker => this.config["Broker"];
private string Topic => this.IsRemote() ? this.FromId().ToString() : this.config["Topic"];
```

### Configurations
In the *GetConfigurationTemplate* method, parse the JSON representation of the settings into the Settings object.
```csharp
var settings = Settings.Parse(template);
new Populator(parameters).Populate(settings);
```
Create a control for the topic and set its value and visibility.
```csharp
var topic = settings.Find("Topic") as TextBox;
topic.Visible = topic.Required = !this.IsRemote();
```

### Validate
When validating the stream, an error needs to be shown if either the topic or the broker fields are left empty. Set the config variable to a new Configuration instance and set the parameters to the parameters received in the *Validate* method.
```csharp
int i = 1;
var errors = new List<string>();
this.config = new Configuration() { Parameters = parameters };

if (String.IsNullOrWhiteSpace(this.Broker))
    errors.Add($"Error {i++}: Broker is not specified.");

if (String.IsNullOrWhiteSpace(this.Topic))
    errors.Add($"Error {i++}: Topic is not specified.");
```

### Create
Set the config variable to the configuration received in the *Create* method.
```csharp
this.config = configuration;
```
Create a new MQTT client, given the value in the *Broker Address* field.
```csharp
this.client = new MqttClient(this.Broker);
```

### Start
There is no need to do anything in the *Start* method.

### Destroy
Make sure that the MQTT client is disconnected in the *Destroy* method.
```csharp
public void Destroy()
{
    if (this.client?.IsConnected == true)
        this.client.Disconnect();
}
```

### Publishing Events
This agent will receive events; thus, the following method has to be implemented:
```csharp
void Receive(string endpointName, JArray events)
```

In the *Receive* method, make sure that the MQTT client is connected. If not, connect to the client, passing a new unique ID.
```csharp
if (this.client.IsConnected == false)
    this.client.Connect(Guid.NewGuid().ToString());
```

Publish the incoming events to the topic that has been subscribed to. Then, publish the events to the *Output* endpoint by invoking the *OnPublish* event.

```csharp
if (this.client.IsConnected == true)
{
    client.Publish(this.Topic, Encoding.UTF8.GetBytes(events.ToString()), MqttMsgBase.QOS_LEVEL_EXACTLY_ONCE, false);
    this.OnPublish?.Invoke(this, new OnPublishArgs(events, "Output"));
}
```

### Decrypting Values
This agent does not use any secure, automatically encrypted, values. There is no need to decrypt anything.
