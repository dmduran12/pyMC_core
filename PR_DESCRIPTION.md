# PR: Radio Watchdog and Recovery System

**Branch:** `feat/radio-watchdog-pr`  
**Target:** `fix/timing`  
**Author:** @dmduran12

---

## Summary

This PR adds a comprehensive radio watchdog and automatic recovery system to address the issue where SX1262 radios can become stuck after TX completion timeout, leaving subsequent CAD/RX operations failing with "tuple index out of range" errors.

## Problem

When running a MeshCore repeater for extended periods, the SX1262 can enter a stuck state, typically triggered by:
1. TX completion timeout ("TX completion timeout - no interrupt received!")
2. The radio fails to return to RX mode properly
3. Subsequent operations fail with cryptic errors

Once stuck, the radio requires a full reset to recover, but without intervention, the repeater becomes deaf to incoming packets.

## Solution

This implementation adds:

### Watchdog State Tracking
```python
self._last_rx_time = time.time()      # Track last successful RX
self._last_health_check = time.time() # Track last health check
self._consecutive_errors = 0          # Error counter
self._max_consecutive_errors = 3      # Recovery threshold
self._rx_timeout_threshold = 120      # 2 minutes without RX triggers concern
self._is_recovering = False           # Prevent concurrent recovery
```

### Health Check API
- `check_radio_health()` - Comprehensive health check that:
  - Restarts dead RX background tasks
  - Detects prolonged RX inactivity
  - Triggers recovery when error threshold is reached

### Automatic Recovery
- `_recover_radio()` - Async recovery with full hardware reset
- `_sync_recover_radio()` - Sync fallback for non-async contexts
- `_hardware_reset_sequence()` - Complete reinitialization:
  1. Clear all interrupts
  2. Hardware reset
  3. Set standby mode with busy-wait verification
  4. Reconfigure all radio parameters (frequency, modulation, packet params, TX power)
  5. Configure interrupts and start RX continuous mode
  6. Reset TX/RX pins to RX mode

### Monitoring Support
- `reset_watchdog()` - Called after every successful RX callback
- `get_health_stats()` - Returns health data for external monitoring:
  ```python
  {
      "time_since_last_rx": float,
      "consecutive_errors": int,
      "is_recovering": bool,
      "rx_timeout_threshold": float,
      "initialized": bool,
      "rx_task_alive": bool,
  }
  ```

### TX Timeout Integration
The TX timeout handler now increments the error counter and can trigger recovery:
```python
# In _handle_transmission_timeout():
self._consecutive_errors += 1
if self._consecutive_errors >= self._max_consecutive_errors:
    await self._recover_radio()
```

## Usage

External applications can integrate health monitoring:

```python
# Periodic health check (e.g., every 10 seconds)
radio.check_radio_health()

# Get health stats for dashboard/API
stats = radio.get_health_stats()
```

## Testing

Tested on:
- LilyGo T3-S3 with E22 SX1262 module
- Extended runtime with intentional stress testing
- Recovery verified after induced TX timeout conditions

## Compatibility

- Builds on top of `fix/timing` branch improvements (IRQ trampoline, GPIO edge handling, etc.)
- No breaking changes to existing API
- New methods are additive and optional to use

## Files Changed

- `src/pymc_core/hardware/sx1262_wrapper.py`
  - Added watchdog instance variables (~lines 146-152)
  - Added `reset_watchdog()` call in RX callback handler
  - Replaced simple `check_radio_health()` with comprehensive version
  - Added `_recover_radio()`, `_sync_recover_radio()`, `_hardware_reset_sequence()`
  - Added `reset_watchdog()`, `get_health_stats()` methods
  - Enhanced `_handle_transmission_timeout()` with error tracking
