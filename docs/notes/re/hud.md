# Stage HUD placement (Rt2d) and widescreen — G9SE8P rev 0

Found 2026-11-15 by static reading of dtk's asm (tools/showfn.py) plus PC RAM dumps in Seaside Hill
(`build/run1/rt_hud169` Fit 1280x720, `rt_hud43` 4:3, `rt_hudpoke`; older `rt_dw` / `rt_dn` from before commit
59c0ca1, when every camera was widened). Names follow RenderWare 3.x's Rt2d toolkit by signature; none are
proven by symbols.

## Summary

The stage HUD (score/time/rings, lives, team icons, Team Blast gauge, pause menu) is drawn with RenderWare's
**Rt2d** toolkit: 2D shapes in front of the camera at a fixed layer depth (3.0), projected by the normal
perspective camera. Every Rt2d draw gets its matrix from one function, `fn_8016F964` [confirmed by reading:
all five Rt2d render paths call it]. That matrix has two parts with different sources:

- **origin (left/top edge)**: the camera's current view window, re-read every frame -> widened in a stage [confirmed];
- **pixel scale** (640x480 HUD pixels -> Rt2d units): baked into the HUD objects' matrices when the HUD loads,
  from the view window Rt2d had then -> 4:3 since the widening is limited to stage cameras [confirmed: values;
  likely: the cause].

So in Fit/16:9 a piece at 4:3 pixel x lands at camera x = depth*(k*w - (2w/640)*px): widened left edge, 4:3 scale.
That is the observed layout: left pieces right, centred pieces (pause menu, special-stage timer) one quarter of
the half-width left of centre, right pieces (team icons, gauge) near the centre-right.

## Rt2d globals `lbl_803EC968` (bss, size 0xA18)

- +0x234 CTM stack (array of RwMatrix pointers), +0x2B4 stack index [likely: push/pop code].
- +0x2B8 = 2 x view window x (`0x803ECC20`), +0x2BC = 2 x view window y, +0x2C0 / +0x2C4 = frame-buffer
  raster width / height as floats (640, 480) [confirmed: written by fn_8016EE88 / fn_8016EF70; dumps: 1.2311 at
  4:3, 1.6428 in Fit, then 1.0, 640, 480].
- +0x2C8 cached device matrix (CTM x depth x flip x origin), +0x308 its inverse, +0x348 the result
  returned to the draw (device matrix x camera frame LTM), +0x388 "device matrix valid" flag (cleared by every
  CTM change), +0x390 layer depth (3.0), +0x3BC current Rt2d camera [likely].

## Functions (main.dol)

| Address | Rt2d name (by signature) | Role | Tag |
|---|---|---|---|
| `fn_8016EF70` | Rt2dOpen(camera) | init; copies camera window x2 and raster size into the globals | likely |
| `fn_8016EE88` | Rt2dDeviceSetCamera(camera) | same copy (store of +0x2B8 at `8016EEFC`, from camera+0x68 x 2.0); called every frame by the HUD drawers (`800C4CC8` in fn_800C4C18, fn_801154BC, fn_800C0EA0) | confirmed (code), likely (name) |
| `fn_8016EBC0` | Rt2dDeviceGetStep(xstep, ystep, origin) | one-pixel steps from +0x2B8/+0x2C0 through the inverse device matrix | likely |
| `fn_8016F964` | (internal) device/draw matrix | builds +0x2C8 from the CTM top, then returns +0x348 = +0x2C8 x camera LTM; called by fn_8016AC30, fn_8016BEDC, fn_8017201C, fn_801751D0, fn_80175720 (the draw paths) | confirmed (code) |
| `fn_8016EAB4` / `fn_8016F894` / `fn_8016F488` / `fn_8016F860` | Rt2dCTMSetIdentity / CTMScale / CTMPush / CTMPop | CTM stack | likely |
| `fn_801733F4` / `fn_8017348C` / `fn_8017349C` / `fn_801734F4` / `fn_80173614` | Rt2dObjectGetMTM / MTMChanged / MTMScale / MTMTranslate / SetVisible-like | object matrix | likely |
| `fn_800A8EC4` (and `fn_800A8D24`) | game: fit a Maestro scene to the screen | GetStep, then object MTM scale = (screen size x step) / scene bbox, translate to centre; screen size from `0x8029BB80` +4/+8 | likely |
| `fn_800C731C` | game: stage HUD loader ("game_disp", string `0x80254330`) | 15 calls of fn_800A8EC4; called at `8001D308` in the stage setup fn_80019034 | likely |
| `fn_800C4C18` | game: per-frame HUD draw | SetCamera(current camera), then the HUD pieces | likely |

## Where the origin and the scale come from

- Origin: `fn_8016F964` loads +0x2B8 (`8016F9F4`) and translates by 0.5 x (2w) x depth (`8016FA68` .. `8016FA88`);
  y likewise from +0x2BC. +0x2B8 is rewritten every frame by `fn_8016EE88` from the current camera, so it follows
  the widened stage window [confirmed: `0x803ECC20` = 1.6428 in Fit, 1.2311 at 4:3].
- Device x unit: the matrix scales by m = min(2w, 2h) (= 1.0 = screen height), not by 2w, so widening does not
  change the size of a unit [confirmed by reading].
- Pixel scale: the HUD objects' matrices hold 2w/640 = 0.0019237 (device units per HUD pixel; camera space
  -3 x that = -0.0057709) and 1/480 for y. 107 copies of 0.0019237 in the Fit dump and in the 4:3 dump; in the old
  all-cameras-widened build (`rt_dw`) the same 107 slots held 0.0025670 (= k x) [confirmed: RAM dumps]. Baked at
  load via GetStep in `fn_800A8EC4` (GetStep calls at `800A8F2C` / `800A905C`), so it takes the window of the camera
  Rt2d had when `game_disp` loaded; since 59c0ca1 that is unwidened [likely: the HUD loads before / outside the
  stage-camera widening].
- Not the source: the sdata2 constant 1.2311481 (`lbl_8042EA44`, `lbl_8042DEC0`, used for CTMScale in
  fn_801154BC and fn_800C4C18's text path). Poking `8042EA44` to the 16:9 value changed nothing in Seaside Hill
  [confirmed: rt_hudpoke shot identical layout].
- Final draw matrices in the Fit dump: x scale -0.0057709 (4:3), translation 2.4642 (= 3 x widened w) for
  pieces at 4:3 pixel 0; team-icon pieces at 4:3 px 472..586, score/rings at 66 / 88 [confirmed: dump]. A few
  matrices (e.g. 0x80454BB0) still carry the 4:3 origin (1.847): probably objects not drawn this frame [uncertain].
- The pause menu on the Switch shot (hw12 6435) sits at 300/800 = the formula for px 320 [likely].

## Fix options (port side, none implemented)

1. Old look (all proportional, icons stretched by k): make the load-time step widened too, e.g. scale the steps
   GetStep returns inside fn_800A8EC4 by k while a stage is being set up (after `800A8F2C` / `800A905C`). The pre-59c0ca1
   build did this implicitly (`rt_dw`: positions spread to the edges, team icons ~k wider, font text not).
2. Consistent 4:3 HUD in the centre: in `fn_8016F964` use the 4:3 window (2w / k) for the origin (`8016FA58` .. `8016FA68`)
   while a stage camera is widened.
3. Edge anchoring without stretch (Fit): per draw, at the tail of `fn_8016F964` (before the multiply at `8016FB54`,
   r4 = +0x2C8), classify the piece by its 4:3 pixel x = (depth*w_cur - M.pos.x) / (depth * 2w43/640) and shift a
   copy's pos.x by 0 (left, < 213), -depth*(w_cur - w43) (centre), -2*depth*(w_cur - w43) (right, > 427).
   Caveat: the translation is the CTM origin; 28 matrices in the dump sit at px 0 (their geometry carries the
   position), so those would always be treated as left [uncertain which pieces they are].
