# auto-safety-hil-ecu

A learning project: a car that learns to drive itself in a 3D simulator, with a real safety computer watching every move.

- **The driver** is a PyTorch model on the PC, which plays the car's onboard computer. It drives like a person, with gas, brake and steering, and sees what a self-driving car's sensors see: lidar, cameras, radar, microphones, GPS, an IMU and wheel speeds.
- **The safety computer** is an STM32 microcontroller on a real car network (CAN bus). It checks every command and steps in when it must: it brakes, or steers away when the car gets too close. It reports every time it does.
- **Each attempt is a run** toward a destination, on a course or a street with pedestrians. Reaching the destination scores well; every time the safety computer had to step in counts against the run. The model learns from the scores and tries different moves next time.
- **Everything is in 3D,** so each run can be watched as it happens, or replayed.

## How data flows

```mermaid
flowchart LR
    subgraph PC["PC (Windows)"]
        World["3D car simulator<br/>(roads, pedestrians, sensors)"]
        Driver["PyTorch driver<br/>(the car's onboard computer)"]
        Score["Run score<br/>(destination reached?<br/>how often did the ECU step in?)"]
    end
    Adapter["USB-to-CAN adapter"]
    Cable(["CAN bus cable<br/>(two twisted wires)"])
    Transceiver["CAN transceiver chip"]
    subgraph STM32["STM32 board (NUCLEO-G474RE, FreeRTOS)"]
        Check{"Check each message:<br/>intact? new? on time? sensible?"}
        Decide["Decide: allow, limit,<br/>brake or steer away"]
        Distrust["Distrust it: slow down safely"]
    end

    World -- "sensor readings" --> Driver
    World -- "sensor summaries, over USB" --> Adapter
    Driver -- "gas, brake, steering, over USB" --> Adapter
    Adapter --- Cable
    Cable --- Transceiver
    Transceiver -- "CAN frames" --> Check
    Check -- "passes" --> Decide
    Check -- "fails" --> Distrust
    Decide -- "final command and any intervention, back over CAN" --> Transceiver
    Distrust -- "final command, back over CAN" --> Transceiver
    Adapter -- "final command, over USB" --> World
    World -- "how the run went" --> Score
    Score -- "learn, then try again" --> Driver
```

The driver on the PC only asks. The STM32 always decides what the car actually does, and every time it steps in, the run scores lower.

> **Not a real safety system.** This is for learning only. It borrows ideas from car safety engineering, but claims no safety standard, and it controls only a simulated car.

## License
MIT, see [LICENSE](LICENSE). Third-party code keeps its own license.
