## .net core ΢����ϵͳ
* 1����������
* 2����־����
* 3��ҵ�����
* 4���������м��
* 5������ע�ᷢ��
* 6�������㷨
* 7���ӷ����໥���������ʹ��5 6��
* 8��΢��������
* 9��ͳһ��֤���ģ�jwt��
* 10���������
* 11����·׷��
* 12���¼�����


## 1����������
*  ���п�����Ŀ������Ϣ:

```csharp
using Bucket.AspNetCore;
public class Startup
{
    //...
    public void ConfigureServices(IServiceCollection services)
    {
        //...

        services.AddConfigService(opt => {
		    opt.AppId = "12313",
            opt.AppSercet = "213123123213",
            opt.RedisConnectionString = "",
            opt.RedisListener = false,
            opt.RefreshInteval = 30,
            opt.ServerUrl = "http://localhost:63430",
            opt.UseServiceDiscovery = false,
            opt.ServiceName = "BucketConfigService"
		}); // << Add this line

        //...
    }
}
```

## 2������ע���뷢��
*  ʹ��Consulʵ�ַ���ע���뷢��:

```csharp
using Bucket.AspNetCore;
using Bucket.AspNetCore.ServiceDiscovery;
public class Startup
{
    //...
    public void ConfigureServices(IServiceCollection services)
    {
        //...

        services.AddServiceDiscovery(option => {
                option.UseConsul(opt =>
                {
                    opt.HostName = "localhost";
                    opt.Port = 8500;
                });
            });

        //...
    }
}
```

* �ӷ���ע�������:

```csharp
using Bucket.AspNetCore;
public class Startup
{
    //...
    private readonly CancellationTokenSource _consulConfigCancellationTokenSource = new CancellationTokenSource();
	public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory, IApplicationLifetime applicationLifetime)
    {

            // add tenant & health check
            var localAddress = DnsHelper.GetIpAddressAsync().Result;
            var uri = new Uri($"http://{localAddress}:{Program.PORT}/");
            var registryInformation = app.AddTenant("values", "1.0.0-pre", uri, tags: new[] { "urlprefix-/values" });
            // register service & health check cleanup
            applicationLifetime.ApplicationStopping.Register(() =>
            {
                app.RemoveTenant(registryInformation.Id);
                _consulConfigCancellationTokenSource.Cancel();
            });
    }
}
```
* �ӷ�����໥����:

```csharp
using Bucket.AspNetCore;
public class Startup
{
    //...
	public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory, IApplicationLifetime applicationLifetime)
    {

         // ʹ�÷����ֵ��ӷ���ӿ�����
         services.AddServiceClient();
    }
}
// ceshi 
private readonly IServiceClient _serviceClient;
public UserService(IServiceClient serviceClient)
{
    _serviceClient = serviceClient;
}
```

## 3���¼�����
* ʹ��RabbitMQʵ���¼�����:

```csharp
using Bucket.AspNetCore;
using Bucket.AspNetCore.EventBus;
public class Startup
{
    //...
    public void ConfigureServices(IServiceCollection services)
    {
        //...

         services.AddEventBus(option=> {
                option.UseRabbitMQ(opt =>
                {
                    opt.HostName = "localhost";
                    opt.Port = 8500;
                    opt.ExchangeName = "BucketEventBus";
                    opt.QueueName = "BucketEvents";
                });
        });

        //...
    }
}
```

## 4��������־
* ʹ���¼�����������Ϣ����:

```csharp
public class Startup
{
    //...
    public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
    {
          var eventBus = app.ApplicationServices.GetRequiredService<IEventBus>();
		  loggerFactory.AddBucketLog(eventBus); // or loggerFactory.AddBucketLog(app);
		  // ��־����
          eventBus.Subscribe<PublishLogEvent, PublishLogEventHandler>();
    }
}
```
* ����̨��ʾ:

```csharp
class Program
    {
        private static IServiceCollection services;
        static void Main(string[] args)
        {
            Console.WriteLine("Hello World!");
            Initialize();

            services.AddEventBus(option => {
                option.UseRabbitMQ(opt =>
                {
                    opt.HostName = "192.168.1.199";
                    opt.Port = 5672;
                    opt.ExchangeName = "BucketEventBus";
                    opt.QueueName = "BucketEvents";
                });
            });
            var eventBus = services.BuildServiceProvider().GetRequiredService<IEventBus>();
            // ��־��ʼ��
            Func<string, LogLevel, bool> filter = (category, level) => true;
            ILoggerFactory loggerFactory = new LoggerFactory();
            loggerFactory.AddBucketLog(eventBus);
            services.AddSingleton(loggerFactory);
            ILogger logger = loggerFactory.CreateLogger<Program>();

            // �¼�����
            eventBus.Subscribe<PublishLogEvent, PublishLogEventHandler>();
            var i = 0;
            while (i < 9)
            {
                i++;
                logger.LogError(new Exception($"���Ǵ�����־{i.ToString()}"), "1");
            }
            Console.ReadLine();
        }
        private static void Initialize()
        {
            var configuration = new ConfigurationBuilder()
                .SetBasePath(Directory.GetCurrentDirectory())
                .AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
                .Build();

            services = new ServiceCollection()
                .AddLogging();
        }
    }
```
![��ͼ](http://10.10.141.47/tianliang/Microservice/blob/master/ConsoleApp2/logimg.png)


## 5��΢��������
* ����Consul + .NET Core + Polly + Ocelot + Exceptionless + IdentityServer

## 6���������
* ����App Metrics

## 7��ͳһ��֤
* ��������ϵͳPOCͳһ����Ȩ����֤��ʹ��Jwt
* ���ز�����û���Ȩ��֤����ɫ��֤��Ȩ����֤
* �ӷ���ʹ����������Ҫ��֤Ȩ�޽ӿڽ���֤�û���Ȩ�������о����ɫȨ����֤��ͨ��HttpContext.User��ȡ��ǰ�����û���Ϣ

```csharp
public class Startup
{
    //...
    public void ConfigureServices(IServiceCollection services)
    {
        // ��Ȩ��֤
        var audienceConfig = Configuration.GetSection("Audience");
        services.AddBucketAuthentication(opt =>
        {
            opt.Audience = audienceConfig["Audience"]; ;
            opt.DefaultScheme = audienceConfig["DefaultScheme"];
            opt.Issuer = audienceConfig["Issuer"];
            opt.Secret = audienceConfig["Secret"];
        });  
    }
}
[Authorize]
public OutputLogin Login([FromBody] InputLogin input){}
```

## 8������������
* �ӿڷ��ض�Ӧ���룬��Ҫ������Ӫרҵ���������

```csharp
public class Startup
{
    //...
    public void ConfigureServices(IServiceCollection services)
    {
        // ����������
        services.AddErroCodeService(opt =>
        {
            opt.RefreshInteval = 300;
            opt.ServerUrl = "http://122.192.33.40:18080";
        });
    }
}
// ģ��
[NotEmpty("001",ErrorMessage = "�˺Ų���Ϊ��")]
public string UserName { set; get; }
// �����뼰��Ӧҵ������
throw new BucketException(errorInfo.ErrorCode, errorInfo.Message);
```

## 9��������
* 

```csharp
public class Startup
{
    //...
    public void ConfigureServices(IServiceCollection services)
    {
        // ������
        services.AddBuriedService();
    }
}
```