# How phones and tablets work

A phone, a tablet, a laptop, and a desktop all use a processor to run software, memory to hold current work, and storage to keep data when the power is off. What changes is the design budget. A phone must stay reachable all day, fit in a pocket, and remain comfortable to hold. A laptop trades some of that portability for a larger screen, keyboard, and often a fan. A desktop tower can draw continuous power from the wall and devote much more space to cooling.

## Inside a phone or tablet

The main chip is usually a **system on a chip (SoC)**. It combines general-purpose CPU cores with processors dedicated to jobs such as drawing graphics, processing camera images, and decoding video. An internal network connects these components to each other and to the memory controller. The diagram shows typical connections. The exact chip boundaries vary by model.

```text
                  Camera, touch, display, microphone, speakers
                                      |
                                      v
                         Device controllers and interfaces
                                      |
  Working memory (RAM) <------> [ Mobile SoC ] <------> Flash storage
                                CPU, GPU, ISP,
                                media engine,
                                AI accelerator
                                      |
                           Connectivity hardware
                             /               \
                    Wi-Fi / Bluetooth    Cellular modem*
                                              |
                                      Radio and antennas

  Battery -> power-management circuits -> chips, display, radios
  * Present on phones and cellular-equipped tablets.
```

| Component | What it does | A job you can observe |
| --- | --- | --- |
| CPU (central processing unit) | Runs the operating system and general-purpose app code. Many designs combine cores tuned for speed with cores tuned for efficiency. | Responds to a tap and decides which screen to open. |
| GPU (graphics processing unit) | Performs many similar calculations in parallel. | Draws a game's 3D scene. |
| ISP (image signal processor) | Processes raw readings from the camera sensor, including color reconstruction and noise reduction. | Turns sensor readings into a recognizable preview. |
| Media engine | Encodes or decodes supported video formats in hardware. | Plays a compressed video while leaving the CPU free for other work. |
| NPU (neural processing unit), where present | Accelerates operations used by supported machine-learning models. | Can run a model that recognizes objects in a photo. |
| RAM and flash storage | RAM holds active code and temporary data. Flash retains apps, photos, and files after shutdown. | A photo being edited occupies RAM. The saved photo survives a restart in flash. |
| Modem and radio hardware | The cellular modem handles the cellular connection. Radio circuits and antennas transmit and receive signals. The modem can be part of the SoC or a separate chip. | Receives a call while the screen is off. |
| Sensors and low-power controllers | Measure motion, orientation, light, or proximity. A low-power controller can monitor inputs while the main CPU sleeps. | Counts steps or turns off the screen when you hold the phone to your ear. |

The CPU, GPU, and accelerators typically have access to the same physical RAM. A camera frame can stay in a shared buffer while different processors work on it. This can save copying, time, and energy. The processors still share a finite memory bandwidth, and software must coordinate when each one can read or write the buffer. Qualcomm's Snapdragon platform provides one concrete example of how these processors fit together. [Snapdragon platform overview](https://www.qualcomm.com/smartphones/products/8-series/snapdragon-8-gen-3-mobile-platform)

## From an app to the hardware

Consider an Android app opening the camera. The arrows show how a **request** travels down the software stack. Captured frames then become available to the authorized app for display or processing.

```text
Camera app
    -> Camera framework API       the app requests a preview
    -> System camera service      the OS checks access and manages the device
    -> Hardware abstraction layer the vendor implements a common camera interface
    -> Kernel drivers             drivers control the hardware and its buffers
    -> Camera sensor and ISP      the device captures and processes the image
```

The **kernel** is the part of the operating system that schedules CPU time, manages memory, and controls access to devices. Android uses a Linux kernel. Its hardware abstraction layer lets the same camera API work across devices with different sensors. iPhones and iPads expose hardware through Apple frameworks and drivers, using the Darwin/XNU kernel. [Android platform architecture](https://developer.android.com/guide/platform)

Two policies affect what an app can do. **Sandboxing** isolates an app's private data, while **permissions** govern access to resources such as the camera. **Background execution** determines when an app can keep working after you switch away. For example, iOS and iPadOS can suspend an ordinary app and give an eligible task time to run later. A desktop app can generally keep computing when its window loses focus and the computer stays awake. [Android application sandbox](https://source.android.com/docs/security/overview/kernel-security) · [Apple background execution strategies](https://developer.apple.com/documentation/backgroundtasks/choosing-background-strategies-for-your-app)

## What happens when you record a video?

1. **Capture.** The camera sensor converts incoming light into digital readings. The ISP processes those readings into frames and can adjust color and reduce noise.
2. **Preview.** The camera pipeline places frames in RAM. The display system combines the preview with the app's controls so you can see what you are recording.
3. **Save.** A hardware video encoder compresses frames in a supported format. The system combines the encoded video with microphone audio and writes the recording to flash.
4. **Manage heat.** During a long recording, the camera, processor, display, and storage all draw power and produce heat. If the device approaches its temperature limits, the system can reduce processing speed or limit camera features. A large cooler can sustain a higher rate of heat removal, which is one reason a desktop can often maintain a heavy workload for longer. [Android thermal management](https://source.android.com/docs/core/power/thermal-mitigation)

## Phones, tablets, laptops, and desktops compared

These are common designs. The desktop column describes a tower PC. Compact desktops and all-in-one computers can have tighter cooling and upgrade limits.

| Design choice | Phone | Tablet | Laptop | Desktop tower |
| --- | --- | --- | --- | --- |
| Processing | Tightly integrated SoC, usually Arm-based | Often a phone-class or laptop-class SoC | Arm or x86-64 processor, often with integrated graphics. Some add a separate GPU. | Commonly x86-64, with integrated graphics or an expansion-card GPU. Arm desktops also exist. |
| Power and cooling | Small battery, typically fanless. The case spreads heat. | Larger case can fit a bigger battery and spread heat across more surface. The larger display also draws power. | Runs on battery or wall power. Fans are common, with some fanless designs. | Wall power, larger heatsinks, and more room for airflow support sustained workloads. |
| Memory and storage | RAM and flash usually fixed at purchase | Similar to phones, with capacities varying by model | RAM may be soldered or socketed. Some solid-state drives are replaceable. | Memory modules, storage drives, and graphics cards are often replaceable independently. |
| Connectivity and input | Cellular, Wi-Fi, touch, cameras, and motion sensors are central to the design. | Wi-Fi is typical, cellular is optional. More screen space supports pens and keyboards. | Keyboard and trackpad are typical. Cellular and touch depend on the model. | External displays, keyboards, and mice are typical. Ethernet and expansion ports are common. |
| Software behavior | Android or iOS manages app permissions and background work. | Android and iPadOS follow mobile app policies. Some tablets run desktop Windows. | Windows, macOS, or Linux commonly supports long-running user programs. | Similar operating systems to laptops, with more scope for internal expansion. |

**Why the CPU label alone is insufficient.** Arm and x86-64 name instruction-set architectures, the machine instructions a CPU understands. An Arm phone and an Arm laptop can understand the same CPU instructions while running different apps because their operating systems provide different APIs and application formats. A tablet running Windows can therefore behave more like a laptop than an Android tablet of the same size. [Microsoft's Windows on Arm overview](https://learn.microsoft.com/en-us/windows/arm/overview)

**How to compare performance.** Opening a photo tests short bursts of work. Exporting a long video tests the encoder, memory bandwidth, storage, and the cooling system over time. Two devices that feel equally quick opening the editor can finish the export at different times. For that comparison, check the hardware support for the chosen video format and the performance each device can sustain.
