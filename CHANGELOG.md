Changelog:

Fix: Two spring-state bugs in the target lock rewrite
-----------------------------------------------------------------------------------------------------------------
  * Spring velocity state was shared between the camera springs (LookAtTarget, driving
    thirdPersonState->freeRotation) and the player rotation springs (UpdateRotationLockedCam,
    driving the player's heading/pitch). The two code paths are mutually exclusive per frame,
    but switching between free-cam lock and locked-cam mode (or mounting/dismounting) carried
    momentum accumulated against one quantity over to the other, producing a visible snap or
    overshoot. The state is now split into separate _cameraYawVelocity/_cameraPitchVelocity
    and _playerYawVelocity/_playerPitchVelocity members (all reset in ResetTargetTracking()).
  * The pitch spring in LookAtTarget is skipped while _isBehind (camera transition behind the
    player). Its velocity was frozen with a stale value during the whole transition and then
    released at once when tracking resumed, kicking the camera pitch at transition end.
    The pitch spring velocity is now zeroed while the spring is skipped.

Improved: Target switch on mouse movement (accumulated-travel detection with hysteresis)
-----------------------------------------------------------------------------------------------------------------
  * The old implementation compared each individual MouseMoveEvent delta against the
    sensitivity threshold using an L1 norm (|x|+|y|). Because a single fast flick
    generates many events that each exceed the threshold, one flick could chain-fire
    multiple switches. The bTargetRecentlySwitched latch was set in the mouse path
    but never actually checked there (dead code), so it did not prevent this.
  * The new implementation accumulates mouse deltas into a virtual stick position
    that decays toward zero with a ~150ms half-life (time-based, using steady_clock).
    A switch fires when the accumulated dominant-axis (Chebyshev) magnitude reaches
    the sensitivity threshold, and re-arms only after the position decays back below
    40% of the threshold (hysteresis).
  * One flick = exactly one switch. Slow, deliberate movement still accumulates and
    triggers. Continued steady movement in one direction cycles targets.
  * uTargetLockMouseSensitivity now maps directly to mouse travel distance required
    per switch, so the setting has a meaningful effect across its whole range.
  * Diagonal movement no longer inflates the trigger distance (Chebyshev instead of
    L1 norm). Sensitivity is clamped to a minimum of 1.
  * MCM help text updated to describe the new behavior.

Rewrite: Target Lock camera tracking replaced with analytic spring (BREAKING - not backwards compatible)
-----------------------------------------------------------------------------------------------------------------
  * The legacy InterpAngleTo + hard clamp target lock implementation has been replaced entirely.
  * No legacy fallback code remains. No master toggle. The new implementation IS the implementation.

  * Bug 1 fixed (whirlwind sprint pass-through fails to lock on):
    - FindTarget adds velocity-based lookahead for fast-moving targets (>fTargetLockFastTargetVelocity).
    - FindTarget extends the lock range by 15% for fast targets.
    - CheckCurrentTarget relaxes the distance check (up to 2x) during a lock grace period
      (fTargetLockLockGraceDuration, default 0.4s) after target acquisition.

  * Bug 2 fixed (lock on at specific height fails / appears clamped):
    - GetCameraAngle uses a soft logistic clamp (SoftClamp) instead of a hard clamp.
    - The camera approaches the pitch limit asymptotically rather than hitting a wall.
    - Tunable via fTargetLockPitchSoftClampWidth (default 0.25 rad) and fTargetLockPitchSoftClampK (default 5).

  * Bug 3 fixed (underwater camera prevention causes shake while swimming + locked):
    - GetSmoothedCameraGroundHeight applies EMA smoothing to the water/land height sample.
    - Eliminates per-frame jitter from waving water surfaces (the swim-shake bug).
    - Tunable via fTargetLockWaterHeightSmoothingRate (default 6).

  * Critically-damped spring (CriticallyDampedSpringAngle in Utils.h):
    - Replaces InterpAngleTo for all camera yaw/pitch and player yaw/pitch tracking.
    - Uses the closed-form analytic solution of the spring ODE (not numerical integration).
    - Genuinely frame-independent: stepping dt=X once produces the exact same result as
      stepping dt=X/N N times. No drift when framerate varies.
    - Velocity is passed by reference and accumulates momentum across frames, allowing
      the camera to briefly overshoot to catch fast-moving targets (the key property that
      distinguishes a spring from a single-pole filter).
    - The 50ms dt cap in LookAtTarget has been removed (the analytic solution is stable at any dt).
    - Tunable via fTargetLockSpringStiffness (omega, default 8 rad/s -> ~0.375s settle).

  * Velocity-based target lookahead:
    - UpdateTargetTracking computes the target velocity each frame (with low-pass smoothing).
    - LookAtTarget leads the target by velocity * lookaheadTime (scales with distance).
    - Tunable via fTargetLockLookaheadTime (default 0.10s) and fTargetLockFastTargetVelocity (default 300).

  * Removed settings (no longer toggleable, always on):
    - bTargetLockModernization, bTargetLockUseSpringDamper, bTargetLockVelocityPrediction,
      bTargetLockSmoothWaterHeight, bTargetLockSoftPitchClamp, bTargetLockLockGracePeriod.

  * Remaining tunable settings (all in MCM under 'Target Lock' page):
    - fTargetLockSpringStiffness (spring omega, default 8)
    - fTargetLockLookaheadTime (lookahead seconds, default 0.10)
    - fTargetLockWaterHeightSmoothingRate (EMA rate, default 6)
    - fTargetLockPitchSoftClampWidth (soft zone width, default 0.25 rad)
    - fTargetLockPitchSoftClampK (soft clamp stiffness, default 5)
    - fTargetLockLockGraceDuration (grace period seconds, default 0.40)
    - fTargetLockFastTargetVelocity (fast target threshold, default 300)

Fix: Access violation crash in DirectionalMovementHandler::AddProjectileTarget (rax=0 in std::_Hash::_Forced_rehash)
-----------------------------------------------------------------------------------------------------------------
  * Root cause: The _projectileTargets unordered_map was being mutated from the projectile aim hook (Hooks::ProjectileHook::ProjectileAimSupport) which runs on a BSJobs worker thread, while simultaneously being iterated/erased on the main thread via UpdateProjectileTargetMap(). std::unordered_map is not thread-safe even for single-writer/single-reader pairs; during a _Forced_rehash triggered by emplace(), the bucket array is reallocated and a concurrent reader on another thread can dereference a stale (null) bucket pointer, producing the observed EXCEPTION_ACCESS_VIOLATION at TrueDirectionalMovement.dll+0038C19 (cmp r9d, [rax+0x10] with rax=0x0).
  * Fix: DirectionalMovementHandler already declared a recursive_mutex _lock and Locker alias but never used them. Added Locker guards in:
      - UpdateProjectileTargetMap()
      - GetProjectileTargetPoint()
      - AddProjectileTarget()
      - RemoveProjectileTarget()
  * Recursive mutex chosen so the lock is safe even if a guarded public API ends up calling another guarded public API on the same thread.

Enable TargetLock when mounted on a dragon:
------------------------------------------

  Hooks.cpp and Hooks.h
  * New class DragonCameraState, same functions and members as HorseCameraState, but for the dragon camerastate
  * New hook DragonCameraStateHook (same functions as HorseCameraStateHook, but hooking into the DragonCameraState)
  * Renamed SaveCamera::RotationType::kHorse to SaveCamera::RotationType::kMount to reflect both horse and dragon mounts (no functional changes)

  DirectionalMovementHandler.cpp and Hooks.cpp
  * Added checks for RE::CameraStates::kDragon whererever there are checks for RE::CameraState::kMount
  * Use DragonCameraState instead of HorseCameraState in case camera state is RE::CameraState::kDragon

  DirectionalMovementHandler.cpp
  * modified handling of camera snap in ToggleTargetLock() and LookAtTarget() such that DragonCameraState->dragonRefHandle.get()->As<RE::Actor>()->data.angle.x is not modified.
  ( otherwise if target lock is on, the dragon's orientation flickers between two positions while the dragon is switching flying states (flying->hover, or hover->land)
  * In ToggleTargetLock(), when toggling the TargetLock on, the target lock is set to the dragon's current combat target instead of using the actor which is closest to the center of the screen
  (only in case IDRC is active and the dragon is mounted and in combat). This functionality uses IDRC's API function APIs::IDRC->GetCurrentTarget(). It is used if APIs::IDRC->UseTarget() returns true.
  * In LookAtTarget(): 
    - use dragon instead of player as reference because player's yaw is changing with dragon's head orientation.

    - enhanced the handling of the situation when the target is behind the camera (ie camera is between player and target):
      - added distance check to bIsBehind to avoid that the camera is rotating towards the player-axis in case the target is further away from the player as the camera
    (in that case, the target ended up behind the camera).  
      - The distance check also addresses this case: If the target is close behind the camera there was a tendency for camera oscillation between two positions close to the player-target axis. Reason is that projectedDirectionToTargetXY changes significantly through the camera movement because the distance btw camera and target is short. That in turn leads to angleDelta values switching sign between frames.

  * IDRC API is added via API/IDRC_API.h, and connected in API/APIManager.cpp and API/APIManager.h 
      * IDRC API requires unreleased version of IDRC (https://github.com/staalo18/IntuitiveDragonRideControl)


Changed handling of reference pitch for horseCameraState and dragonCameraState:
-------------------------------------------------------------------------------
  * DirectionalMovementHandler.cpp: in LookAtTarget(), use referencePitch = 0 for horse and dragon camera states, instead of _desiredPlayerPitch.


New option: TargetLock - Min height above ground:
-------------------------------------------------
  * Introduced functionality to keep the camera above mininmal height over ground. Two new functions:
    * DirectionalMovementHandler.cpp: GetCameraAngle() -  Checks for ground level and adapts angle if camera would move below min height.
    *  Utils.cpp: GetLandHeightWithWater() - provides z-coord of land height, considering water level.
  *  New MCM option  (in TargetLock category): "Min Camera Height Above Ground", with default 35. This required changes in:
      MCM/Config/TrueDirectionalMovement/config.json
      MCM/Config/TrueDirectionalMovement/settings.ini
      Translation/TrueDirectionalMovement_english.txt
      Settings.cpp
      Settings.h
  * In Settings.h: bTargetLockConsiderGroundLevel (true) - switch to turn off Min-height-above-ground functionality

New option: TargetLock - Lock behind target:
--------------------------------------------
  * DirectionalMovementHandler:
    * new functions:
        EnableLockBehindTarget)
        ToggleLockBehindTarget()
        GetNominalCameraToPlayerDistance()
        GetNominalCameraPosition()
        UpdateMoveCameraBehindTarget()
    * LookAtTarget(): Modifications to compute yaw and pitch for the "lock-behind-target" case (probably more bad math ahead ...). 
  * Utils.cpp: new function GetFlyingState()
  * Settings.h: fCameraBehindTargetMinDistance, fCameraBehindTargetNoSwitchRange (not in MCM)
  * New MCM options (in TargetLock category): "Enable Lock Behind Target" (default: off), and "Toggle Lock-Behind-Target Key" (default: undefined). This required changes in:
      MCM/Config/TrueDirectionalMovement/config.json
      MCM/Config/TrueDirectionalMovement/settings.ini
      Translation/TrueDirectionalMovement_english.txt
      Settings.cpp
      Settings.h
      Events.cpp (in InputEventHandler::ProcessEvent() - check for Toggle Lock-Behind-Target Key)
