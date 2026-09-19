# cpu-powersave

Cap CPU power and frequency on Linux, persistently.

A home server spends most of its life idle and the rest doing something
unhurried. Left alone, the CPU still boosts to its maximum clock at the
slightest provocation - spending watts and making heat to finish a task two
seconds earlier that nobody was waiting for.

This script turns that off and keeps it off. It detects your hardware, asks
what you want capped, and installs a systemd service that reapplies the
settings at every boot.

```
$ cpu-powersave

Hardware
  cpu           Intel(R) Core(TM) Ultra 9 185H
  cores         22  (intel_pstate)
  frequency     12 cores  base 2300  max 5100 MHz
                 8 cores  base 1800  max 3800 MHz
                 2 cores  base 1000  max 2200 MHz
  turbo         on
  power limit   45 W  (hardware maximum 64 W)

Questions
  press enter to accept the value in brackets

  Turbo boost, on or off [off]: off
  Frequency floor, e.g. 2.0GHz (empty = none): 2.0GHz
  Frequency ceiling (empty = none):
  Energy preference: 1) default  2) performance  3) balance_performance  4) balance_power  5) power [4]:
  Power limit in watts, max 64 (0 = leave alone) [45]: 35

To apply
  turbo         off
  floor         2000 MHz
  ceiling       none
  preference    balance_power
  power         35 W over 28s
  with turbo off every core is capped at its base clock

  Apply and enable at boot? [Y/n] y

┌─ [1/3] Settings
│  file          /etc/default/cpu-powersave
└─ ✓ written, read at every boot

┌─ [2/3] Boot service
│  helper        /usr/local/sbin/cpu-powersave-apply
│  service       /etc/systemd/system/cpu-powersave.service
└─ ✓ enabled and started, the settings return at every boot

┌─ [3/3] Check
│  turbo         off
│  frequency     12 cores  2000-2300 MHz
│                 8 cores  1800 MHz (pinned)
│                 2 cores  1000 MHz (pinned)
│  preference    balance_power
│  power         35 W
└─ ✓ everything applied, and reapplied at every boot
```

Each question comes with one plain line saying what it changes, left out
above for brevity. The last step reads everything back from the hardware: the
firmware has the final say on turbo and the power limit, and when it kept its
own value the step ends with `!` and says which one.

## Install

```bash
git clone https://github.com/Lousblue/cpu-powersave.git
sudo install -m 755 cpu-powersave/cpu-powersave /usr/local/bin/
```

No dependencies. It is one bash script talking to `/sys`.

## Use

```bash
cpu-powersave                  # ask, showing what was detected
cpu-powersave --eco            # a profile, no questions
cpu-powersave --no-turbo --min 2.0GHz --power 35
cpu-powersave --status         # read only, no privileges needed
cpu-powersave --eco --dry-run  # show what a profile would do, change nothing
cpu-powersave --revert         # undo everything
```

Pass any setting and it stops asking - which is also what happens with no
terminal, so it is safe to call from a script or a provisioning tool.

### Profiles

| | turbo | floor | preference | power |
|---|---|---|---|---|
| `--eco` | off | half the base clock | `power` | 70% of the current limit |
| `--balanced` | off | none | `balance_power` | unchanged |
| `--performance` | on | none | `performance` | unchanged |

### Settings

| Option | |
|---|---|
| `--turbo on\|off`, `--no-turbo` | turbo boost |
| `--min FREQ`, `--max FREQ` | frequency bounds - `2.0GHz`, `2000MHz`, `2,5` with a comma, or a bare number guessed by size |
| `--epp NAME` | energy performance preference |
| `--power WATTS` | package power limit, `0` to leave it alone |
| `--tau SECONDS` | window that limit averages over, default 28 |
| `--dry-run` | show what would be applied, then stop |
| `--yes` | never ask: turbo off, nothing else |

## What it changes

Five knobs, in this order - turbo first, because the driver recomputes the
ceilings it reports the moment turbo changes.

1. **Turbo boost.** The one that matters. Off, every core is capped at its
   base clock, which is where most of the saving comes from.
2. **The `powersave` governor**, written straight to sysfs.
3. **Frequency bounds**, clamped per core.
4. **The energy performance preference**, which tells the hardware governor
   how eagerly to clock up on a short burst.
5. **The package power limit** (Intel RAPL), integrated graphics included.
   This is the only setting that bounds the whole package rather than the
   cores alone.

### On hybrid CPUs

Performance, efficient and low-power cores do not share a ceiling. A floor of
2.0 GHz is impossible on a core that tops out at 1.0 GHz, so rather than
refuse it, the script pins that core at its own ceiling and says so:

```
  frequency     12 cores  2000-2300 MHz
                 8 cores  1800 MHz (pinned)
                 2 cores  1000 MHz (pinned)
```

## Files it writes

| | |
|---|---|
| `/etc/default/cpu-powersave` | your settings, read at every boot |
| `/usr/local/sbin/cpu-powersave-apply` | the helper that applies them |
| `/etc/systemd/system/cpu-powersave.service` | runs the helper at boot |

To change a setting, either rerun the script or edit the first file and
`systemctl restart cpu-powersave.service`.

`--revert` removes all three and restores the firmware defaults, including the
original power limit and governor, which it saved the first time you installed
- reinstalling with different values keeps that first snapshot.

## Hardware

Anything with a `cpufreq` driver, which is every modern x86 CPU and most ARM
boards. What differs between vendors:

| | Intel (`intel_pstate`) | AMD (`amd_pstate`) |
|---|---|---|
| Frequency bounds | yes | yes |
| Energy preference | yes | yes, with `amd_pstate_epp` |
| Turbo | `intel_pstate/no_turbo` | `cpufreq/boost` |
| Package power limit | yes, RAPL | **no** - read only on AMD |
| `--eco` floor | yes | no, it needs `base_frequency` |

**Tested on** an Intel Core Ultra 9 185H (Meteor Lake, hybrid) running Proxmox
VE 9 on Debian 13, and against simulated Intel and AMD sysfs trees in a
container.

**Not tested on real AMD hardware.** The code paths exist and behave correctly
against a simulated `amd_pstate` tree, but nobody has run this on an actual
Ryzen. `--status` will tell you what it found before you change anything, and
`--revert` undoes it. Reports welcome.

Where a setting cannot be applied - a firmware that locks the power limit is
common on mini PCs - the script says so rather than failing silently.

## Caveats

- **Your CPU will be slower.** That is the point, but a host that transcodes
  video or compiles will notice. Measure before you keep it.
- **Other power managers will fight it.** TLP, `tuned` and
  `power-profiles-daemon` write to the same knobs; whichever runs last wins.
  Run one or the other, not both.
- **`--revert` restores the power limit** it saved, but if the write is
  refused the cap clears at the next reboot.
- Writing to `/sys` needs root. `--status` does not.

## Licence

MIT.
