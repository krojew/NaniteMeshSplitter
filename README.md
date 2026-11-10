# Nanite Mesh Splitter

An Unreal Engine editor plugin that splits a static mesh by material. The opaque sections stay on the original
mesh, which gets Nanite turned on. Masked, translucent and decal sections move to a second mesh next to it, with
Nanite off.

## Why split a mesh

Kit and prop meshes often come merged: one container mesh carries the painted metal, the trim decals, the warning
signs and the light covers. Nanite only renders Surface-domain materials with an Opaque or Masked blend mode, so a
single decal or translucent section keeps the whole mesh off Nanite.

Masked materials do render under Nanite, through its programmable rasterizer, which costs more than opaque
geometry. The plugin always moves masked sections to the non-Nanite mesh. That suits trim, decals and small cutouts;
there is no setting to keep them on Nanite.

Doing this by hand means exporting the mesh, separating it in a DCC tool and importing two meshes. The plugin edits
the mesh where it is and keeps your LOD, collision and section settings.

## Installation

Install the plugin from Fab, or copy the `NaniteMeshSplitter` folder into your project's `Plugins` folder. Then
open **Edit > Plugins**, enable **Nanite Mesh Splitter** and restart the editor.

The plugin is editor-only. Nothing from it ships in a packaged game.

## Usage

1. In the Content Browser, right-click one or more static meshes.
2. Choose **Split Non-Opaque Sections**.
3. Check both meshes, then save them. Nothing is saved for you.

The new mesh is named after the original with a `_NonOpaque` suffix, for example `SM_Container_NonOpaque` next to
`SM_Container`. If that name is taken, a number is added to it.

Place the new mesh wherever the original is used, at the same transform. Actors, Blueprints and levels are not
changed: until you add the new mesh, they show only the opaque part.

When several meshes are selected, meshes that cannot be split are skipped and the Output Log says why (log category
`LogNaniteMeshSplitter`).

## What counts as opaque

A section stays on the Nanite mesh when its material:

- is in the **Surface** domain,
- has an **Opaque** blend mode, counting a material instance's blend mode override,
- uses a shading model Nanite supports (Single Layer Water is not one).

An empty material slot renders with the engine's default material, which is opaque, so it stays too.

Everything else moves to the new mesh: Masked, Translucent, Additive, Modulate and the other translucent blend modes,
Deferred Decal materials and any other non-Surface domain.

## What the split changes

**The original mesh**

- keeps its opaque sections and loses the rest,
- has Nanite turned on,
- keeps its simple collision, LOD settings and build settings. The moved sections take their complex collision
  with them.

**The new mesh**

- holds every non-opaque section,
- has Nanite turned off,
- has no simple collision shapes, because the original keeps the same ones,
- has no complex collision either if the original takes it from a separate Complex Collision Mesh,
- otherwise keeps the per-section settings of its sections, such as collision, shadow casting and ray tracing
  visibility.

So the two meshes do not collide twice. One difference remains: once Nanite is on, the engine builds the
original's complex collision from its Nanite fallback mesh, which can be coarser than the source geometry. If the
mesh uses complex collision and its detail matters, raise the fallback's triangle percentage in the Nanite settings.

**Both meshes**

- keep only the material slots their sections use. Static mesh components override materials by slot index, so
  check placed actors that override materials on the original.
- are split the same way at every imported LOD. A LOD cannot be empty, so an imported LOD left without sections
  on one of the meshes is removed from that mesh, together with the LODs after it. That mesh then keeps drawing
  its last remaining LOD at those distances.
- get their generated LODs rebuilt from the split base LOD, with the existing reduction settings. Section settings
  made on a generated LOD itself are taken from its base LOD again. A LOD that is reduced from another LOD counts
  as generated even if it still stores imported geometry, because the build does not use that geometry.

Reimporting either mesh from its source file brings back the unsplit geometry.

**Undo** after the menu action restores the original mesh. The new mesh stays; delete it if you do not want it.

## Meshes that are skipped

| Reason | What it means |
|---|---|
| Every section is opaque | Nothing needs to move, and Nanite is left as it is. Turn it on in the Static Mesh Editor if you want it. |
| No section is opaque | Nothing would be left for the Nanite mesh, so the mesh is left as it is. |
| No source geometry | The mesh has no source data in this project, so there is nothing to split. |
| Nanite hi-res source | Not supported. Remove the hi-res source, or split the mesh in a DCC tool. |
| Nanite assembly | Not supported. |
| A LOD reduces its own imported geometry | Make that LOD reduce from another LOD (its Base LOD setting) first. |

LOD 0 decides whether a mesh is split. Sections with no triangles are ignored.

## Scripting

The same split is available to Python and to Editor Utility Blueprints through `UNaniteMeshSplitterLibrary`:

| Function | Does |
|---|---|
| `SplitStaticMesh(Mesh) -> (Result, NonOpaqueMesh)` | Splits the mesh, as the menu entry does. |
| `CanSplitStaticMesh(Mesh) -> Result` | Reports what a split would do, without changing anything. |
| `IsOpaqueMaterial(Material) -> bool` | Applies the rule above to one material. |
| `GetResultText(Result) -> Text` | A one-sentence explanation of a result. |

```python
import unreal

mesh = unreal.load_asset("/Game/Props/SM_Container")
result, non_opaque = unreal.NaniteMeshSplitterLibrary.split_static_mesh(mesh)
if result == unreal.NaniteMeshSplitResult.SPLIT:
    unreal.EditorAssetLibrary.save_loaded_assets([mesh, non_opaque], only_if_is_dirty=False)
else:
    unreal.log_warning(unreal.NaniteMeshSplitterLibrary.get_result_text(result))
```

A script call can be undone only if the script runs it inside an editor transaction, as the menu action does.

## Support

Report problems at https://github.com/krojew/NaniteMeshSplitter/issues.
