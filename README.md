# Tank Battle 3D Unity

## Description

**Genre:** Action / Local Multiplayer

**Tank Battle** is a 3D game for two players developed in Unity. Each player controls a tank in an urban arena and competes in rounds until one of them reaches the required number of victories to win the match.

The objective is simple and straightforward: destroy your opponent's tank before they destroy yours. The match is divided into rounds, and the first player to win five rounds becomes champion. Between rounds, the tanks are repositioned and the battle begins again.

The main mechanic revolves around a progressive charging firing system. By holding down the fire button, the player accumulates power in the projectile — releasing the button at the right moment is fundamental to hitting the target and causing maximum damage. The projectile explosion has a radius of effect, which means a shot near the opponent can still cause damage even without a direct hit.

---

## Game

### Gameplay Video

[![Gameplay of the Game](https://github.com/user-attachments/assets/5cd24d72-6fcd-451e-bac2-edb4b82352f8)]([https://github.com/user-attachments/assets/78520674-a691-4957-8ba2-f1bce2f2a723](https://github.com/user-attachments/assets/5cd24d72-6fcd-451e-bac2-edb4b82352f8))

### Game Screenshots

#### Main Menu

![Main Menu](/Images/tanks.png)


#### Gameplay

![Gameplay 1](/Images/playing1.png)

![Gameplay 2](/Images/playing2.png)

---

## Controls

The game supports two players simultaneously on the same keyboard.

### Player 1

| Action       | Key             |
|--------------|-----------------|
| Move forward | `W`             |
| Move backward | `S`             |
| Turn left    | `A`             |
| Turn right   | `D`             |
| Charge / Fire | `Left Ctrl` (hold to charge, release to fire) |

### Player 2

| Action       | Key             |
|--------------|-----------------|
| Move forward | `Up Arrow`      |
| Move backward | `Down Arrow`    |
| Turn left    | `Left Arrow`    |
| Turn right   | `Right Arrow`   |
| Charge / Fire | `Enter` (hold to charge, release to fire) |

### How to Play

1. Upon startup, wait for the round countdown.
2. Move around the map and position yourself strategically.
3. Hold down the fire button to accumulate power in your shot — an indicator on screen displays the charging level.
4. Release the button to fire. The longer you hold, the greater the power and range of the projectile.
5. The round ends when only one tank remains active.
6. The first player to win 5 rounds wins the match.

---

## Implemented Features

### Feature 1: Progressive Charging Firing System

**How it works:**

When you press the fire button, the player initiates the loading of the projectile's power. The power increases progressively while the button is held, until it reaches the maximum value at which point the shot fires automatically. Releasing the button before that causes the projectile to be launched with the accumulated power up to that moment. A slider in the interface visually indicates the charging level in real time.

**File:** `Assets/_Completed-Assets/Scripts/Tank/TankShooting.cs`

```csharp
private void Update ()
{
    // The slider displays the minimum force by default
    m_AimSlider.value = m_MinLaunchForce;

    // If the maximum force has been reached and the shot has not been fired yet...
    if (m_CurrentLaunchForce >= m_MaxLaunchForce && !m_Fired)
    {
        // ...use the maximum force and fire automatically.
        m_CurrentLaunchForce = m_MaxLaunchForce;
        Fire ();
    }
    // If the fire button has just been pressed...
    else if (Input.GetButtonDown (m_FireButton))
    {
        m_Fired = false;
        m_CurrentLaunchForce = m_MinLaunchForce;

        // Play the charging sound.
        m_ShootingAudio.clip = m_ChargingClip;
        m_ShootingAudio.Play ();
    }
    // If the button is being held and the shot has not been fired...
    else if (Input.GetButton (m_FireButton) && !m_Fired)
    {
        // Increment the force and update the slider.
        m_CurrentLaunchForce += m_ChargeSpeed * Time.deltaTime;
        m_AimSlider.value = m_CurrentLaunchForce;
    }
    // If the button has been released and the shot has not yet been fired...
    else if (Input.GetButtonUp (m_FireButton) && !m_Fired)
    {
        Fire ();
    }
}

private void Fire ()
{
    m_Fired = true;

    // Instantiates the projectile at the position and rotation of the tank barrel.
    Rigidbody shellInstance =
        Instantiate (m_Shell, m_FireTransform.position, m_FireTransform.rotation) as Rigidbody;

    // Applies velocity to the projectile in the direction the tank is pointing.
    shellInstance.velocity = m_CurrentLaunchForce * m_FireTransform.forward;

    // Plays the firing sound.
    m_ShootingAudio.clip = m_FireClip;
    m_ShootingAudio.Play ();

    // Resets the launch force.
    m_CurrentLaunchForce = m_MinLaunchForce;
}
```

**Feature screenshot in action:**

![Firing Feature](/Images/action1.png)

---

### Feature 2: Explosion Damage with Distance-Based Calculation

**How it works:**

When the projectile collides with any object, it detonates and checks all tanks within an explosion radius using `Physics.OverlapSphere`. The damage caused to each tank is calculated proportionally to the distance: tanks at the center of the explosion receive maximum damage, while those at the edge of the radius receive minimum damage. In addition to damage, a physical impulse force is applied to the hit tanks, pushing them away from the point of impact. Each tank's health bar is updated in real time and changes color from green (full health) to red (critical health).

**File:** `Assets/_Completed-Assets/Scripts/Shell/ShellExplosion.cs`

```csharp
private void OnTriggerEnter (Collider other)
{
    // Collects all colliders within the explosion radius, filtering only tanks.
    Collider[] colliders = Physics.OverlapSphere (transform.position, m_ExplosionRadius, m_TankMask);

    for (int i = 0; i < colliders.Length; i++)
    {
        Rigidbody targetRigidbody = colliders[i].GetComponent<Rigidbody> ();

        if (!targetRigidbody)
            continue;

        // Applies explosion force physically to the tank.
        targetRigidbody.AddExplosionForce (m_ExplosionForce, transform.position, m_ExplosionRadius);

        TankHealth targetHealth = targetRigidbody.GetComponent<TankHealth> ();

        if (!targetHealth)
            continue;

        // Calculates and applies damage based on distance.
        float damage = CalculateDamage (targetRigidbody.position);
        targetHealth.TakeDamage (damage);
    }

    // Unlinks and plays explosion particles.
    m_ExplosionParticles.transform.parent = null;
    m_ExplosionParticles.Play();
    m_ExplosionAudio.Play();

    ParticleSystem.MainModule mainModule = m_ExplosionParticles.main;
    Destroy (m_ExplosionParticles.gameObject, mainModule.duration);
    Destroy (gameObject);
}

private float CalculateDamage (Vector3 targetPosition)
{
    // Vector from explosion to target.
    Vector3 explosionToTarget = targetPosition - transform.position;

    // Distance between explosion and target.
    float explosionDistance = explosionToTarget.magnitude;

    // Proportion of maximum distance (how close to center the target is).
    float relativeDistance = (m_ExplosionRadius - explosionDistance) / m_ExplosionRadius;

    // Damage proportional to maximum possible damage.
    float damage = relativeDistance * m_MaxDamage;

    // Ensures damage is never negative.
    damage = Mathf.Max (0f, damage);

    return damage;
}
```

**Feature screenshot in action:**

![Explosion and Damage Feature](/Images/action2.png)

---

## Project Structure

```
Assets/
├── _Completed-Assets/
│   └── Scripts/
│       ├── Camera/
│       │   └── CameraControl.cs       # Dynamic camera control with automatic zoom
│       ├── Managers/
│       │   ├── GameManager.cs         # Round management, victories and game flow
│       │   └── TankManager.cs         # Individual tank management
│       ├── Shell/
│       │   └── ShellExplosion.cs      # Explosion logic and damage calculation by distance
│       ├── Tank/
│       │   ├── TankHealth.cs          # Tank health and death system
│       │   ├── TankMovement.cs        # Movement and engine audio
│       │   └── TankShooting.cs        # Firing system with progressive charging
│       └── UI/
│           └── UIDirectionControl.cs  # UI element orientation control
├── AudioClips/                        # Engine, firing and explosion sounds
├── Materials/                         # Tank and scenario color materials
├── Models/                            # 3D models (tanks, buildings, terrain)
└── Scripts/                           # Base version of scripts (for study)
```

---

## Requirements to Run

- Unity 2020.3 LTS or higher
- Operating system: Windows, macOS or Linux

### How to open the project

1. Clone or download the repository.
2. Open Unity Hub and click **Add project from disk**.
3. Select the project root folder.
4. Open the main scene located in `Assets/_Completed-Assets/Scenes/`.
5. Click **Play** in the editor to test.

---

## Technologies Used

- **Unity** — development engine
- **C#** — programming language for scripts
- **Unity Physics (Rigidbody)** — physics simulation for tanks and projectiles
- **Unity Particle System** — visual effects for explosion and smoke
- **Unity Audio Mixer** — sound mixing for ambient, engine and explosions
- **Unity UI (uGUI)** — interface for health bar and firing slider
