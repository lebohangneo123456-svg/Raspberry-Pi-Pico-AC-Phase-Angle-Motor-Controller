# ACS307B — RASPBERRY PI PICO PHASE-ANGLE CONTROLLER V3 - FASTER PROFILE
# Proteus 8 / MicroPython — 12 V AC TEST CIRCUIT ONLY
# GP14 = H11AA1 zero crossing input (HIGH near each zero)
# GP15 = 2N2222 -> MOC3021 optotriac input
# No PWM / no DC-DC converter.
#
# TEST IN TWO STEPS:
# 1. FIX THE MOTOR-BRIDGE DC- TO MOTOR- DIRECT CONNECTION.
# 2. Confirm correct 50 Hz crossing pulses, then enable firing.
# 3. Test FIXED_ANGLE_DEG = 35 for fast operation; use None for auto.

from machine import Pin
from time import ticks_us, ticks_ms, ticks_diff, sleep_us

FIRE_ENABLED = True           # Enable firing for low-voltage simulation after wiring check
FREQUENCY_HZ = 50
HALF_CYCLE_US = 1000000 // (2 * FREQUENCY_HZ)
GATE_PULSE_US = 1500
LOOP_SLEEP_US = 200
CYCLE_MS = 20000
ANGLE_LOW = 35
ANGLE_HIGH = 95
REPORT_MS = 2000
# For a constant firing-angle test, set FIXED_ANGLE_DEG = 35.
# Set back to None for the automatic 20-second project cycle.
FIXED_ANGLE_DEG = None

zc = Pin(14, Pin.IN, Pin.PULL_UP)
fire = Pin(15, Pin.OUT)
fire.value(0)


def angle_for_time(elapsed_ms):
    if FIXED_ANGLE_DEG is not None:
        return FIXED_ANGLE_DEG
    t = elapsed_ms % CYCLE_MS
    half = CYCLE_MS // 2
    span = ANGLE_HIGH - ANGLE_LOW
    if t < half:
        return ANGLE_HIGH - (span * t // half)
    return ANGLE_LOW + (span * (t - half) // half)


def firing_delay_us(angle):
    return angle * HALF_CYCLE_US // 180


print('ACS307B: PICO PHASE CONTROL V3 - FASTER PROFILE')
print('GP14 = ZERO CROSS, GP15 = OPTOTRIAC DRIVER')
print('GATE FIRING ENABLED:', FIRE_ENABLED)

initial_ms = ticks_ms()
last_report_ms = initial_ms
last_level = zc.value()
high_started_us = None
last_valid_us = None
valid_streak = 0
zero_count = 0
fire_count = 0
late_count = 0

# 0 = idle; 1 = waiting to fire; 2 = gate pulse active
state = 0
schedule_started_us = 0
schedule_delay_us = 0
gate_started_us = 0

while True:
    now_us = ticks_us()
    now_ms = ticks_ms()
    level = zc.value()

    # H11AA1 output rises near zero crossing.
    if level == 1 and last_level == 0:
        high_started_us = now_us

    # Falling edge terminates the HIGH window.
    elif level == 0 and last_level == 1:
        if high_started_us is not None:
            high_width = ticks_diff(now_us, high_started_us)
            if last_valid_us is None:
                since_last = HALF_CYCLE_US
            else:
                since_last = ticks_diff(now_us, last_valid_us)

            valid = (150 <= high_width <= 6000 and
                     7000 <= since_last <= 15000)

            if valid:
                last_valid_us = now_us
                zero_count += 1
                valid_streak += 1
                angle = angle_for_time(ticks_diff(now_ms, initial_ms))
                # Estimate the zero-crossing midpoint. The falling
                # edge occurs roughly half a HIGH-window after zero.
                delay = firing_delay_us(angle) - high_width // 2
                if delay < 150:
                    delay = 150
                    late_count += 1

                fire.value(0)
                state = 0
                if FIRE_ENABLED and valid_streak >= 4:
                    schedule_started_us = now_us
                    schedule_delay_us = delay
                    state = 1
            else:
                valid_streak = 0
        high_started_us = None

    last_level = level

    if state == 1 and ticks_diff(now_us, schedule_started_us) >= schedule_delay_us:
        fire.value(1)
        gate_started_us = now_us
        fire_count += 1
        state = 2

    elif state == 2 and ticks_diff(now_us, gate_started_us) >= GATE_PULSE_US:
        fire.value(0)
        state = 0

    # Missing zero crossings => gate output OFF.
    if last_valid_us is not None and ticks_diff(now_us, last_valid_us) > 35000:
        fire.value(0)
        state = 0
        valid_streak = 0

    if ticks_diff(now_ms, last_report_ms) >= REPORT_MS:
        angle = angle_for_time(ticks_diff(now_ms, initial_ms))
        print('ANGLE:', angle,
              'ZC/2s:', zero_count,
              'GATES/2s:', fire_count,
              'LATE:', late_count)
        if zero_count == 0:
            print('CHECK U3 AC INPUTS / GP14')
        elif zero_count < 160 or zero_count > 240:
            print('CHECK ZERO-CROSS WIDTH / TIMING')
        last_report_ms = now_ms
        zero_count = 0
        fire_count = 0
        late_count = 0

    sleep_us(LOOP_SLEEP_US)
