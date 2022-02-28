---
class: 'text-center'
highlighter: shiki
lineNumbers: false
info: |
  ## Watering herbs with Raspberry Pi - Smart irrigation system made with sensors and Azure IoT Central
drawings:
  persist: false

layout: image-right
image: https://blog.novacare.no/content/images/2022/02/20220131_165308-3.jpg
---
# Når man er for lat til å vanne 🌧️

Et selvvanningssystem bygget på Raspberry Pi, .NET 6 og Azure IoT Central

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->

---

# Motivasjon

Å bygge noe fra bunnen av er gøy og veldig givende

- 👶 **Pappapermen** - Flere uker uten å røre tastaturet kan knekke en hver mann
- 🔩 **Utstyr som støver ned** - Raspberry Pi og annen elektronikk til overs fra andre prosjekter
- 💰 **Kjøper en del urter** - Mye urter blir kjøpt for en relativt dyr sum
- ✍🏻 **Upløyd mark** - Nesten ingen guider eller bloggposter om å lage dingser med .NET
- 🧠 **Lære nye ting** - Lære nye sider av .NET og Azure

---

# Handleliste

- 🖥️ **Maskin** - Raspberry Pi 3B+
- 📈 **Sensorer** - DS18B20 temperatur sensor (med 4.7 Ohm resistor) og jordfuktighetssensor (med MCP3008 ADC)
- 🚂 **Motorer** - Vannpumper og lamper kontrollert av relé med 4 kanaler
- 🛠️ **Bygg** - Brødbrett, ledninger, vannrør, vanntank og hobbyplater

---

# Hvordan skal dette virke?

Hvert 30. minutt sjekker sensorene to ting, jordfuktighet og jordtemperatur. 

➡️ Når jordfuktigheten er for lav, utløser vi vannpumpen i noen sekunder.

➡️ Når jordtemperaturen er for lav, utløser vi lampen til vi når den ideelle temperaturen.

---
layout: image-right
image: https://blog.novacare.no/content/images/2022/02/20220130_220648.jpg
---

# La oss smelle sammen en prototype!

- Installére Raspberry Pi OS
- Aktivére 1-Wire og SPI (Serial Peripheral Interface)
- Installére .NET 6 `curl -sSL https://dot.net/v1/dotnet-install.sh | bash /dev/stdin --channel Current`
- Koble alt i hop!

---

# Cronjob

Jeg valgte å kjøre applikasjonen som en worker basert på dotnet-labs sitt eksempel: `*/30 * * * *`.

```csharp
using System.Reflection;
using herbinator;

using var host = Host.CreateDefaultBuilder(args)
    .UseContentRoot(Path.GetDirectoryName(Assembly.GetEntryAssembly()?.Location))
    .ConfigureServices((hostContext, services) =>
    {
        services.AddCronJob<PlantJob>(c =>
        {
            c.TimeZoneInfo = TimeZoneInfo.Local;
            c.CronExpression = @"*/30 * * * *";
        });
    })
    .Build();

await host.RunAsync();
```

---

# Datamodell

Records er tynt og ikke muterbart ❤️

```csharp
using Iot.Device.OneWire;

public record Plant(int Id,
  string ConnectionString,
  string Name,
  SoilMoisture SoilMoisture,
  Thermometer Thermometer,
  int PumpGpio,
  int LightGpio);

public record Thermometer(string DeviceId, string BusId, double Max)
{
    public OneWireThermometerDevice Device => new(BusId, DeviceId);
}
public record SoilMoisture(int Channel, double MoistureThreshold);
```

---

# Plantekonfigurasjon

```json
{ 
  "plants":[
    {
      "id": 0,
      "name": "Basil",
      "connectionString": "<connectionstring>",
      "soilMoisture": {
        "channel": 0,
        "moistureThreshold": 400
      },
      "thermometer": {
        "deviceId": "28-86010c1e64ff",
        "busId": "w1_bus_master1",
        "max": 25
      },
      "pumpGpio": 19,
      "lightGpio": 26
    }
  ]
}
```

---

# Håndtering av vanningen

```csharp
private Task<double> HandleSoilMoisture(Plant plant)
{
    var hardwareSpiSettings = new SpiConnectionSettings(0, 0);
    using var spi = SpiDevice.Create(hardwareSpiSettings);
    using var mcp = new Mcp3008(spi);
    double soilMoisture = mcp.Read(plant.SoilMoisture.Channel);
    if (soilMoisture > plant.SoilMoisture.MoistureThreshold)
    {
        using GpioController controller = new();
        controller.OpenPin(plant.PumpGpio, PinMode.Output);
        controller.Write(plant.PumpGpio, PinValue.High);

        Thread.Sleep(2000);

        controller.Write(plant.PumpGpio, PinValue.Low);
    }

    Console.WriteLine($"Soil moisture for {plant.Name}: {soilMoisture}");
    return Task.FromResult(soilMoisture);
}
```

---

# Håndtering av lyset

```csharp
private async Task<double> HandleThermometer(Plant plant)
{
    var thermometerDevice = plant.Thermometer.Device;
    var temperature = await thermometerDevice.ReadTemperatureAsync();
    if (temperature.DegreesCelsius > plant.Thermometer.Max)
    {
        using GpioController controller = new();
        controller.OpenPin(plant.LightGpio, PinMode.Output);
        controller.Write(plant.LightGpio, PinValue.Low);
    }
    else
    {
        using GpioController controller = new();
        controller.OpenPin(plant.LightGpio, PinMode.Output);
        controller.Write(plant.LightGpio, PinValue.High);
    }

    Console.WriteLine($"Temperature for {plant.Name}: {temperature.DegreesCelsius:F2}°C");
    return temperature.DegreesCelsius;
}
```
---

# Sende data til Azure IoT Central

```csharp
private async Task SendMsgIotHub(Plant plant, double degreesCelsius, double soilMoisture)
{
    var deviceClient = DeviceClient.CreateFromConnectionString(plant.ConnectionString, TransportType.Mqtt);
    var telemetry = new Telemetry(degreesCelsius, soilMoisture);
    var eventMessage = new Message(Encoding.UTF8.GetBytes(JsonSerializer.Serialize(telemetry)));
    await deviceClient.SendEventAsync(eventMessage);
}
```
---

# Kjøre applikasjonen på Raspberry Pi

Når applikasjonen er klar til å kjøre, publiserer vi den med en `linux-arm` runtime: `dotnet publish -r linux-arm`.

For å få applikasjonen til å kjøre ved oppstart, må vi opprette og registrere en systemd tjeneste.

```
[Unit]
Description=Herbinator Water Monitoring
After=network.target

[Service]
ExecStart=/home/pi/path/to/project/herbinator

[Install]
WantedBy=network.target
```

Nå trenger vi bare å registrere tjenesten med `sudo systemctl daemon-reload` og `sudo systemctl enable herbinator.service`.

---
layout: image
image: https://blog.novacare.no/content/images/2022/02/20220110_200911.jpg
---

---
layout: image
image: https://blog.novacare.no/content/images/2022/02/20220131_161803.jpg
---

---
layout: image
image: https://blog.novacare.no/content/images/2022/02/20220131_164147.jpg
---

---
layout: image
image: https://blog.novacare.no/content/images/2022/02/20220131_165308-3.jpg
---

---

# Noen erfaringer rikere

- Lampene ga ikke fra seg så mye varme... De er så og si alltid på.
- IKKE la vanntanken stå høyere opp enn plantepottene!
- Kontaktlim under vann er ikke en god idé
- Kjøp alltid flere deler enn man trenger
- Kjøp fra butikker med god returpolicy

---