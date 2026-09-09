The complete implementation is in `pushout_explicit.py`, divided into ten numbered sections. It generates the input deck, imports an orphan-mesh model into Abaqus/CAE when available, defines an input-file job, runs Abaqus/Explicit, and extracts load–slip and energy histories after successful completion.

The Python generator has been executed and checked locally. **Abaqus is unavailable in this environment: the CAE import, solver data check, analysis and real ODB postprocessing have not been executed.** The supplied material and reinforcement data are illustrative, not a calibrated prediction of specimen capacity.

## Coordinate system and assembly

Lengths are in mm. X runs across the slab width, Y is normal to the flange–concrete interfaces, and Z points upward. The origin is midway between the slabs at their support elevation. Loading acts in −Z. Units are N, mm, MPa, s and tonne; steel density is therefore 7.85 × 10⁻⁹ tonne/mm³.

| Component | X bounds | Y bounds | Z bounds |
|---|---:|---:|---:|
| Positive-Y concrete slab | −300 to 300 | 87.5 to 187.5 | 0 to 650 |
| Negative-Y concrete slab | −300 to 300 | −187.5 to −87.5 | 0 to 650 |
| Positive-Y flange | −45 to 45 | 79.5 to 87.5 | 100 to 750 |
| Negative-Y flange | −45 to 45 | −87.5 to −79.5 | 100 to 750 |
| Steel web | −2.5 to 2.5 | −79.5 to 79.5 | 100 to 750 |
| Positive-X stiffener | 2.5 to 22.5 | −4 to 4 | 675 to 725 |
| Negative-X stiffener | −22.5 to −2.5 | −4 to 4 | 675 to 725 |

The 650 mm steel length is retained. Raising its bottom to Z = 100 puts its top at Z = 750, exactly 100 mm above the slabs. Its lower end remains clear of the supports during the default 10 mm push. The stiffener location in the exposed steel is an editable assumption; its 8 × 20 × 50 mm dimensions are retained.

## Bolt arrangement and length interpretation

The specified 2 × 2 pattern on **each** slab would require eight bolts. To retain four bolts total, the default uses this arrangement:

| Slab | Upper bolt, Z = 500 | Lower bolt, Z = 250 |
|---|---:|---:|
| Positive Y | X = +20 | X = −20 |
| Negative Y | X = −20 | X = +20 |

This arrangement has 180° rotational symmetry about Z. It is not mirror symmetric across each individual coordinate plane. Set `BOLT_LAYOUT = 'EIGHT_TOTAL'` to place X = ±20 bolts at both elevations on both slabs.

The script interprets 130 mm as **shank length**, with an additional 5 mm head. If the intended 130 mm includes the head, change `BOLT_SHANK_LENGTH` to 125 mm. Define local t as the outward distance from a flange–concrete interface:

| Region | Local t bounds, mm |
|---|---:|
| Concrete | 0 to 100 |
| Steel flange | −8 to 0 |
| Bolt shank | −60 to 70 |
| Bolt head | −65 to −60 |
| Nut | −13 to −8 |
| Blind concrete hole | 0 to 70.5 |

Thus 70 mm of shank is inside concrete, 8 mm crosses the flange, and 52 mm projects behind it. The nut bears against the flange; the head is remote from the flange. Both a seated head and this shank length/embedment would require another geometric detail. This choice is recorded in the script rather than shortening the bolt silently.

Each bolt is one deformable solid mesh containing separate SHANK, HEAD and NUT element sets. Shared nodes represent perfect attachment between them. The nut is an annulus around the shank; overlapping solid nut/shank volumes are avoided. Threads, pretension, adhesive, grout and expansion-anchor action are not modelled. The resulting connection represents a smooth dowel transferring shear through bearing and friction, not a calibrated bonded or mechanical anchor pullout law.

## Script sections

1. Geometry, material, reinforcement, friction, mesh and analysis parameters.
2. Parameter checks and deterministic quadrilateral/brick mesh generation.
3. Concrete slabs, blind holes, supports and local slip gauges.
4. I-section, stiffeners and integrated shank/head/nut assemblies.
5. B31 reinforcement wire meshes and clearance checks.
6. Input-deck parts, sections, assembly and materials.
7. General Contact and support conditions.
8. Explicit step, Smooth Step displacement and output requests.
9. ODB extraction of load–slip curves and energy diagnostics.
10. Geometry audit, CAE import, job definition and execution.

## Mesh and interactions

All solids use C3D8R. Circular boundaries are polygonal approximations with 16 segments by default. Mapped rings surround holes, and square cores avoid collapsed elements at bolt axes. The blind-hole backfill shares nodes with the surrounding concrete. Refinement controls include circle segments, radial layers, axial spacing and bulk spacing; changing diameter or embedment regenerates geometry and mesh together.

The I-section is one part with flange and web mesh regions joined by weld ties. Separate stiffeners are tied to the web. Concrete–steel interfaces and bolt bearing surfaces remain in frictional contact. General Contact includes the exterior of solid elements; reinforcement is deliberately excluded from that contact domain. The approach follows the documented ability to define a selected general-contact domain. [Abaqus General Contact](https://docs.software.vt.edu/abaqusv2025/English/SIMACAEITNRefMap/simaitn-c-contactgeneral.htm)

The illustrative reinforcement is a single orthogonal mat in each slab: diameter 10 mm, 150 mm nominal spacing in both directions, 25 mm minimum surface cover, and mat centre at depth 50 mm. Vertical and horizontal families lie at depths 45 and 55 mm. The horizontal grid has an explicit 25 mm Z offset to clear the bolt holes. The generator rejects layouts that intersect holes or violate cover. Multiple mat centres can be supplied, but their physical separation must also be checked.

Each reinforcement part is embedded in its corresponding concrete host using `*EMBEDDED ELEMENT`. Translational compatibility is imposed without artificially fixing beam rotations. Concrete is not removed under embedded bars, so the usual overlapping host/reinforcement approximation applies. [Embedded element definition](https://docs.software.vt.edu/abaqusv2025/English/SIMACAEKEYRefMap/simakey-r-embeddedelement.htm)

Both slab bases have U3 = 0. One bottom corner per slab fixes U1/U2, and a second fixes U2, removing in-plane rigid motion with limited restraint. This is idealized full-base support, not an explicitly meshed bearing plate. The loading reference point is coupled to the steel top and guided vertically, with its transverse translation and rotation fixed. Web nodes already dependent on flange weld ties are excluded from the coupling node set to avoid duplicate kinematic constraints.

## Materials and outputs

Concrete uses CDP with editable elasticity, compression hardening, tensile softening and separate tension/compression damage tables. Compression input uses inelastic strain. Tensile input uses cracking displacement, with a nonzero residual tensile stress. Compression softening remains mesh dependent; displacement-based tensile softening alone does not establish mesh convergence. The CDP viscosity parameter is zero and should not be treated as an Explicit stabilization control. [CDP definitions and data conventions](https://docs.software.vt.edu/abaqusv2025/English/SIMACAEMATRefMap/simamat-c-concretedamaged.htm), [CDP viscosity parameter](https://docs.software.vt.edu/abaqusv2025/English/SIMACAEKEYRefMap/simakey-r-concretedamagedplasticity.htm)

Steel, bolts and reinforcement use separate elastic/plastic materials. Plastic tables contain stress versus plastic strain. Their values, as well as concrete strengths and damage curves, must be replaced with specimen data for quantitative work.

Field output includes S, LE, PE, PEEQ, U, V, A, RF, STATUS, DAMAGET, DAMAGEC and PEEQT, with contact stress/displacement output. Mises is calculated from S in Visualization; no separate MISES field is necessary. STATUS does not represent concrete damage, and no element-deletion law is enabled. PEEQT and PEEQ describe tensile/compressive plastic evolution in CDP; damage contours are not discrete crack geometry. [Explicit tensor and invariant output](https://docs.software.vt.edu/abaqusv2025/English/SIMACAEOUTRefMap/simaout-c-expoutputvar.htm)

History output includes RP U3/RF3, individual local slip-gauge displacements, ALLIE, ALLKE, ALLAE, ALLWK, ALLVD, ALLFD and ETOTAL.

## Running

Place the script in a writable directory and run the following with your Abaqus launcher:

```text
abaqus cae noGUI=pushout_explicit.py
```

This creates `PushOut_run`, writes `PushOut.inp` and `mesh_audit.json`, imports `PushOutModel.cae`, defines the input-file job, submits the analysis and waits for it. After successful completion it writes the CSV files below. `SUBMIT_JOB = True` is the default. The original input deck is authoritative; CAE imports are orphan meshes and may have keyword import limitations. The job reads the original deck directly. [JobFromInputFile](https://docs.software.vt.edu/abaqusv2025/English/SIMACAEKERRefMap/simaker-c-jobfrominputfilepyc.htm)

For an Abaqus data check only:

```text
abaqus cae noGUI=pushout_explicit.py -- --datacheck
```

This uses a separate job name, `PushOut_check`. Review solver warnings, untied/unembedded-node diagnostics and initial-contact adjustments. A data check does not establish analysis stability.

For generation without submission:

```text
abaqus cae noGUI=pushout_explicit.py -- --build-only
```

The input deck can also be generated with ordinary Python:

```text
python pushout_explicit.py --build-only
```

To generate and analyse using Abaqus Python without the CAE kernel:

```text
abaqus python pushout_explicit.py
```

That path launches the solver command directly and does not create a CAE database. Set `ABAQUS_COMMAND` to the local launcher name if required. To extract an existing ODB without rebuilding:

```text
abaqus python pushout_explicit.py --post PushOut_run/PushOut.odb
```

Choose a new `RUN_DIRECTORY` or `JOB_NAME` for each study. The script refuses to overwrite an existing analysis ODB or locked job. Use a fresh CAE session when regenerating the same model name.

## Load–slip and quasi-static review

`load_slip.csv` contains signed RF3, positive-downward applied load (−RF3), loading-point travel, each local slip, and their mean. Local positive-downward slip is **U3_concrete − U3_steel**, measured at matched initial positions beside each hole. RP travel also contains specimen deformation; it is not identical to local interface slip. The reported load is the total specimen force, not force per bolt or per slab.

`energies.csv` contains the energy histories and kinetic/artificial-to-internal-energy ratios. `energy_review.json` records maximum ratios and the fraction of evaluated kinetic-energy samples below the editable 5% screening target. Evaluation omits early time and states with near-zero internal energy to avoid meaningless ratios.

The initial trial applies 10 mm over 0.5 s with Smooth Step amplitude, without mass scaling. Repeat with a longer step and compare load–slip curves, oscillations and energies. Inspect artificial energy, deformed meshes and damage localization, then repeat with finer contact meshes. A low kinetic-energy ratio by itself does not validate the model or its calibration.

## Verification performed here

`validation.json` records three successful generator checks:

| Case | Nodes including RP | Elements |
|---|---:|---:|
| Default four bolts | 48,077 | 38,940 |
| Eight bolts | 60,049 | 48,476 |
| Four bolts, diameter 18 mm, embedment 80 mm, 32 circle segments | 87,653 | 72,730 |

Checks covered Python syntax, positive brick Jacobians at element centres and all eight corners, analytical polygonal slab/bolt volumes, absence of unintended internal slab surfaces, head/shank/nut mesh connectivity, rotational symmetry, rebar clearance and CDP input conversion screening. These are generator checks, not solver validation or evidence of quasi-static response.
