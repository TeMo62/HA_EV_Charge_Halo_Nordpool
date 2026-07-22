# Operations and restart

## Normal operation

The plan is built at 23:00 when tomorrow's 96 quarter-hour prices are
available. One continuous block is selected separately within each day and
night window. At the start of a block, the correct current limit is set through
OCPP and the charger is made available.

When stopping, `switch.ev_charge_control` is turned off first. After 60
seconds, `switch.ev_charger_available` is turned off. This order is important
because the Halo normally does not accept availability being disabled while
charging is in progress.

## Restarting Home Assistant

V1 intentionally has no automatic restart fail-safe. The OCPP integration and
charger entities become available some time after Home Assistant starts, which
makes a fixed delay unreliable.

If the vehicle is connected, charging may therefore start during a restart.
Check the charger manually after OCPP becomes available again. To stop
charging:

1. Turn off `switch.ev_charge_control`.
2. Wait until charging has stopped.
3. Turn off `switch.ev_charger_available`.

## Changes after a plan has been built

Changing the energy target, current, or time window does not automatically
recalculate an existing plan. Run `automation.ev_build_charge_plan` manually
after such a change if it should apply to the next session.
