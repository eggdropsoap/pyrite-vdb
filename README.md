# Pyrite VDB

A Rusty port of OpenVDB and NanoVDB.

Pyrite VDB is a fork of [OpenVDB-rs][openvdb-rs], with the initial goal of overhauling the API surface from heavily C++-accented Rust into idiomatic and ergonomic Rust.

[Pyrite](https://en.wikipedia.org/wiki/Pyrite) is a sulphur oxidate of iron with large cubic crystals and a gold-like lustre. It seemed fitting.

## VDB: Sparse volumes for every occasion

**VDB** is a sparse volumetric tree data structure and algorithms initially detailed in [Museth (2013), "VDB: High-resolution sparse volumes with dynamic topology"][museth-2013], originally developed to improve on volumetric data structures used in the animated film industry. If you've seen a computer-animated film in the last decade, you've likely seen VDB in action.

The data structure is designed for efficient memory footprint, dynamic topology, and a practically unbounded index space, with fast (amortized O(1) time complexity) reads, writes, insertions, and deletions under random-access, sequential, and stencil access patterns. VDB's suite of algorithms support efficient operations on level sets, signed distance fields, and density volumes for a variety of applications: accelerated ray marching, scan conversion, meshing, constructive solid geometry, animation, numerical analysis and simulation.

The [paper][museth-2013] is really worth a read.

  [museth-2013]: https://dl.acm.org/doi/10.1145/2487228.2487235

## Pyrite VDB's lineage

* [**OpenVDB**][openvdb] is the evolving family of C++ implementations of the original VDB data structure and algorithms.
    * [**NanoVDB**][nanovdb] is the linearized, pointer-free, GPU-memory-friendly variant of the OpenVDB data structure developed by Ken Museth while at Nvidia.
* [**OpenVDB-rs**][openvdb-rs] is the "faithful cover" version of OpenVDB and NanoVDB's C++ implementations, reimplemented in Rust by co-authors [Philippe Llerena](https://github.com/doubleailes) and [Claude Code](https://github.com/claude). This is for the most part a straight port of the bulk of the C++ implementation, and therefore largely shares the original's C++-flavoured code structure and API.
* **Pyrite VDB** is this fork of Llerena & Claude's Rust translation.

  [openvdb]: https://github.com/AcademySoftwareFoundation/openvdb
  [nanovdb]: https://github.com/AcademySoftwareFoundation/openvdb/tree/master/nanovdb/nanovdb
  [openvdb-rs]: https://github.com/OxidePicturesAlliance/vdb

## A modest todo list

In no particular order:

- [ ] Overhaul API surface into idiomatic Rust
- [ ] Full docs coverage
- [ ] Deduplication of code via expanded Trait usage
- [ ] Absorb `tools` module into the types: associate, member, and Trait functions instead of bare functions
- [ ] Strongly-typed algorithms interface
- [ ] Pure Rust tests
- [ ] Attempt transition to dual-licensing MIT & Apache 2.0 [for the usual very good reasons](https://bevy.org/learn/quick-start/plugin-development/#licensing)

# Original OpenVDB-rs README

## vdb

A Rust port of [OpenVDB](https://github.com/AcademySoftwareFoundation/openvdb), the sparse
volumetric data structure and toolset used throughout film VFX.

The port follows the C++ library closely: same tree layout (Root → 32³ → 16³ → 8³ leaves),
same file format, same algorithms. Most tools are checked against numbers produced by the
C++ library (OpenVDB 10) on identical inputs, and file I/O is also verified against
OpenVDB 13.1.

```rust
use pyrite_vdb::prelude::*;
use pyrite_vdb::tools::level_set_sphere::create_level_set_sphere;

// A narrow-band level set of a sphere: radius 1, centered at the origin,
// voxel size 0.05, half-width 3 voxels.
let sphere: FloatGrid = create_level_set_sphere(1.0, Vec3f::zero(), 0.05, 3.0, true)?;
println!("{} active voxels", sphere.active_voxel_count());

vdb::io::write_file("sphere.vdb", &[sphere.into()], &MetaMap::new())?;
let (_meta, grids) = pyrite_vdb::io::read_file("sphere.vdb")?;
```

### Cargo features

All features are on by default.

| Feature | Enables |
|---------|---------|
| `read` | Reading `.vdb` files and streams (`io`), and `.nvdb` files with `nanovdb` |
| `write` | Writing `.vdb` files and streams, `.nvdb` files with `nanovdb`, and `Film::save_ppm`. Every function that creates or writes files is behind this feature |
| `nanovdb` | The `nanovdb` module |

To only read files, and to guarantee the crate contains no code that writes to disk:

```toml
[dependencies]
vdb = { version = "0.1", default-features = false, features = ["read"] }
```

Without `read` and `write` the `io` module is not built and the `flate2` and `lz4_flex`
dependencies are not pulled in.

### What's ported

| Area | Rust module | Notes |
|------|-------------|-------|
| Core math | `math` | `Vec2/3/4`, `Coord`, `CoordBBox`, `BBox`, `Mat3/4`, `Quat`, `Ray`, `Dda`, stats/histograms |
| Transforms | `math::maps`, `math::transform` | All linear maps and the frustum map; serialization is byte-identical to C++ |
| Finite differences | `math::finite_difference`, `math::stencils`, `math::operators` | All schemes (CD/FD/BD, WENO5, HJWENO5); world-space operators via map Jacobians |
| Tree | `tree` | `LeafNode`, `InternalNode`, `RootNode`, `Tree<T>`, cached `ValueAccessor`s |
| Grids & metadata | `grid`, `metadata` | `Grid<T>`, `AnyGrid`, grid class / vector type, typed `MetaMap` |
| File I/O | `io` | `.vdb` read/write, versions 220–225, zip and pure-Rust blosc, half floats, all mask-compression schemes |
| Level sets | `tools::level_set_*`, `signed_flood_fill`, `change_background`, `prune` | Sphere, Platonic solids, box, capsules/tubes, dilated meshes; measure (area, volume, genus); utilities |
| CSG & compositing | `tools::composite`, `tools::tree_ops` | Union / intersection / difference, comp min/max/sum/mul/div/replace, tree merge |
| Topology | `tools::morphology`, `activate`, `clip`, `mask`, `topology_to_level_set` | Dilation and erosion with every neighbour pattern and tile policy |
| Sampling | `tools::interpolation`, `tools::grid_transformer` | Point / box / quadratic (+ staggered) samplers, resampling |
| Filtering & evolution | `tools::filter`, `level_set_filter`, `level_set_tracker`, `level_set_advect`, `level_set_morph`, `volume_advect`, `fast_sweeping` | |
| Grid operators | `tools::grid_operators` | Gradient, curl, divergence, Laplacian, mean curvature, CPT, magnitude, normalize |
| Meshing | `tools::mesh_to_volume`, `tools::volume_to_mesh` | Adaptive meshing gives identical points and polygons to C++ |
| Particles & points | `tools::particles_to_level_set`, `point_index_grid`, `point_partitioner`, `point_scatter`, `points_to_mask`, `volume_to_spheres` | Scattered points are bit-identical to C++ |
| Rays | `tools::ray_intersector`, `tools::ray_tracer` | Level-set and volume intersection, a small renderer |
| Solvers | `math::conj_gradient`, `tools::poisson_solver` | Preconditioned conjugate gradient |
| Point data | `points` | `PointDataGrid` with attribute arrays, codecs, groups and strings; conversion, counting, sampling, scattering, rasterization, advection; C++-compatible file I/O |
| NanoVDB | `nanovdb` | Pointer-free GPU-friendly layout, OpenVDB↔NanoVDB conversion (incl. Fp4/8/16/N, index grids), `.nvdb` I/O, samplers, HDDA; converted grids are byte-identical to C++ |
| Diagnostics | `tools::diagnostics`, `count`, `statistics`, `find_active_values` | |

**Not planned:** OpenVDB AX (needs an LLVM JIT), the Houdini/Maya plugins, Python bindings.

**Known gaps:** the frustum point rasterizer, delayed loading and memory-mapped files, interrupters, non-standard tree
configurations (tools assume the standard 5-4-3 tree).

### Examples

```sh
cargo run --release --example vdb_print -- -l tests/data/sphere.vdb   # like OpenVDB's vdb_print
cargo run --release --example level_set_sphere
cargo run --release --example mesh_roundtrip -- torus.obj 0.01 0.0
cargo run --release --example ray_trace_sphere                         # writes a PPM image
cargo run --release --example nanovdb_sphere
```

### Testing

```sh
cargo test
# Reduced feature sets build and test too, e.g. read-only:
cargo test --no-default-features --features read
```

Tests include:

- **Model-based tree tests:** random edits checked against a dense reference.
- **C++-written files:** reference `.vdb` files written by C++ OpenVDB 10.0.1 in `tests/data`.
  The suite also passes with these files regenerated by OpenVDB 13.1.0, and 13.1.0 reads every
  file the Rust writer produces (see [`tests/data/cpp/README.md`](tests/data/cpp/README.md)).
- **Embedded C++ results:** numbers from C++ runs of the same operations, stored in the tests.

The C++ programs that produced the reference data are in `tests/data/cpp/`. With them
built, `VDB_REF=<path to vdb_ref> cargo test --test io -- --ignored` also checks that C++
reads every file the Rust writer produces.

### Benchmarks

`benches/compare.sh` runs the same operations with this crate and with C++ OpenVDB 10 and
prints a comparison table; see [`benches/README.md`](benches/README.md) for the benchmark list
and the full results.

Across the 33 benchmarks, this crate is now faster than C++ OpenVDB 10 overall: the geometric
mean of Rust time ÷ C++ time is **0.88× on one thread** and **0.94× on 4 threads** (it was
1.14× and 1.42× before the performance pass). Every benchmark's result check (voxel counts,
value sums, mesh sizes, ray hits, point counts) matches C++; only the sizes of written files
differ slightly, because the compressors are different implementations.

Selected rows (ratios below 1 mean Rust is faster):

| benchmark | 4 threads: before → now | 1 thread: before → now |
|---|---:|---:|
| `particles_to_sdf` | 3.25× → 0.99× | 1.95× → 0.81× |
| `point_index_grid` / `point_data_grid` | 2.6× / 3.1× → 0.91× / 0.94× | 2.1× / 2.0× → 0.72× / 0.99× |
| `grid_copy` | 3.90× → 0.66× | 1.91× → 1.12× |
| `csg_union` / `csg_difference` | 2.76× / 2.65× → 0.78× / 1.00× | 1.52× / 1.16× → 1.03× / 0.93× |
| `filter_gaussian` | 1.60× → 1.00× | 1.81× → 1.00× |
| `ls_filter_mean_curvature` | 1.41× → 1.00× | 1.52× → 1.00× |
| `gradient` / `laplacian` / `mean_curvature` | 0.83–1.27× → 0.58–0.91× | 0.87–1.06× → 0.52–0.61× |


To run the bench in pixi:
```bash
pixi run bench-compare                       # all benchmarks
pixi run bench-compare --threads 1 --reps 3  # single-threade
```


What made the difference:

- **Tree core:** small node masks (including every leaf's) are stored inline instead of in a
  separate allocation; tree copies and voxel counts run in parallel; lookups test the child
  mask first, so each tree level costs one memory read.
- **Accessors and stencils:** an accessor finds a voxel's leaf or tile in one descent and
  caches the last tile; stencils, grid operators and box samplers read neighbors inside the
  same leaf straight from its buffer.
- **Rounding:** `Coord::floor`/`round`/`ceil` no longer call libm's `floor`/`ceil`, which the
  baseline x86-64 target cannot inline.
- **Parallelism:** particle rasterization, CSG, sphere creation, mesh voxelization and
  volume-to-mesh combine their per-thread results in parallel. Where the work split changed,
  the output was checked to be identical to the sequential result (particle rasterization
  has a test comparing parallel and single-threaded output exactly).
- **Algorithms:** point partitioning uses a stable counting sort instead of a comparison sort.

Still slower than C++:

- `coherent_read` (about 1.8–2×).
- Random-access reads: `random_read`, `box_sample` and `ray_intersect` (1.2–1.5×), which are
  limited by memory latency. Storing each internal-node slot as a single child-or-value
  entry, as C++ does, would likely help, but is a larger refactor.
- Meshing and `mesh_to_level_set` on 4 threads (1.1–1.25×), and `sphere_create`,
  `signed_flood_fill` and adaptive meshing on one thread (1.1–1.5×).

The numbers come from a shared 4-vCPU cloud VM, where single rows move by 10–20% between
runs; the geometric means are more reliable. Benchmarks run on their own (with a name filter)
are also slower than in a full run, because their memory comes fresh from the operating
system, so compare full runs of `benches/compare.sh`.

### License

MIT (see `LICENSE`). OpenVDB itself is licensed under Apache-2.0 / MPL-2.0; this is an
independent port.
