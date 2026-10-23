# Upstream drafts

**Status:** drafts, 2026-10-23. **Nothing has been posted.** Morgan posts, or
decides not to.

---

## 1. pyigtl (lassoan/pyigtl): POLYDATA vertices packed in native byte order; POLYDATA unpack broken

**Title:** PolyDataMessage: VERTICES written little-endian on x86/arm64, and `_unpack_content` raises on any POLYDATA

**Body:**

Two problems in `PolyDataMessage`, found with pyigtl 0.3.4 while testing an
OpenIGTLink peer.

1. **Packing.** `_pack_content` writes the vertex cell array with
   ```python
   binary_content += self.vertices.astype(np.int32).tobytes()
   ```
   which uses native byte order. Lines, polygons and strips (the following
   lines) use `np.ascontiguousarray(..., dtype='>I')`, which is big-endian as
   OpenIGTLink requires. So on little-endian machines every vertex index is
   byte-swapped on the wire: a receiver reads vertex `4` as `67108864`. It also
   requires `vertices` to be a numpy array, while the other arrays accept
   lists. Suggested fix:
   ```python
   binary_content += np.ascontiguousarray(self.vertices, dtype='>I').tobytes()
   ```

2. **Unpacking.** `_unpack_content` (commented "This method has not been
   tested") does
   ```python
   poly_data_header = struct.Struct('> I I I I I I I I I I')
   number_of_points = poly_data_header[0]
   ```
   which indexes the `Struct` object instead of unpacking `content`, so it
   raises `TypeError` for every incoming POLYDATA, inside the client's receive
   thread. Suggested fix: `poly_data_header = struct.unpack_from('> 10I', content)`
   and `start_position = 40`.

A self-contained decoder we used in tests, for reference (OpenIGTLink v1/v2
header, then the ten-field POLYDATA header, points, and four cell arrays),
is in `ros2_igtl_bridge/test/test_polydata.py` (`_read_polydata`).

---

## 2. openigtlink/ros2_igtl_bridge: offer of fixes

**Title:** Offer: Jazzy/Lyrical/Rolling support, shutdown crash fix, POLYDATA port, tests and CI

**Body:**

Thank you for the ROS 2 bridge. We needed it on current ROS 2 for
robot-assisted TMS (SlicerRoboTMS on ROS 2), and have a branch with these
changes on top of `78bc280`. Each is a separate commit, so you can take any
subset:

1. **Use-after-free on shutdown.** The OpenIGTLink thread is detached and
   loops on `this`, so after `main()` returns it runs on a destroyed node, and
   `igtl_node` exits with SIGSEGV on every Ctrl-C. The fix makes the thread
   joinable, with a stop flag, and polls the socket before each header read,
   so stop is noticed within 200 ms. (A receive timeout would be unsafe:
   `igtl::Socket::Receive` drops a partial header on timeout.) It also stops
   logging "Waiting for connection" at ERROR level every second.
2. **Humble through Rolling without warnings.** Uses
   `rosidl_get_typesupport_target` (as in PR #3, without the distro-name
   comparison) and imported targets instead of `ament_target_dependencies`. It
   includes OpenIGTLink as SYSTEM headers and switches to the `tf2_ros/*.hpp`
   headers.
3. **POLYDATA ported to ROS 2**, converting directly between
   `igtl::PolyDataMessage` and the message, with no VTK. The ROS 1 converter
   negated x and y of incoming points only; the port passes them through and
   offers a `legacy_polydata_xy_flip` parameter.
4. **Launch tests** over real TCP with pyigtl as the peer (TRANSFORM, STRING,
   POLYDATA, both directions, clean exit), and a GitHub Actions matrix for
   Humble, Jazzy, Lyrical and Rolling.

Branch: https://github.com/m9h/ros2_igtl_bridge/tree/jazzy *(link valid once
published)*. Happy to open PRs in whatever split you prefer.

---

## 3. OpenRoboTMS/SlicerRoboTMS: ROS 2 branch and three notes

**Title:** ROS 2 support, and three small discrepancies

**Body:**

Thank you for publishing SlicerRoboTMS. Since ROS 1 Noetic reached end of
life in May 2025, we made a `ros2` branch that works with a maintained ROS 2
OpenIGTLink bridge, and wrote down the OpenIGTLink contract the modules use
(`docs/IGTL-CONTRACT.md`). Three things we noticed:

1. The README and CHANGELOG say status is exchanged as OpenIGTLink `STATUS`
   messages; the modules read text nodes, which travel as `STRING`.
2. `Config.py` loads `urdfs/fr3.urdf`, but the repository has no `urdfs/`
   directory, so the robot model cannot load from a fresh clone.
3. `Navigation.py`'s comment says the target x axis is world X projected;
   the code uses head +Y.

Also, the ROS 1 bridge negated x and y of incoming POLYDATA points, which
`Registration.startRegistration` pre-compensates. The ROS 2 bridge passes
points through, so the branch sends them unchanged.

Is the integration layer (tracking, calibration, registration, FR3 control)
planned for release? We are writing a ROS 2 one against the same contract,
and would rather align with yours than diverge.
