# stable-diffusion-webui

AUTOMATIC1111's Gradio 3.41 web UI and FastAPI REST API for Stable Diffusion (SD1/2, SDXL, SD3): txt2img, img2img/inpaint, "Extras" postprocessing (upscalers, face restore), embedding/hypernetwork training, and a plugin system (scripts and extensions). Target is Python 3.10. 3.11 works. 3.12+ breaks the pinned deps (Pillow 9.5, old torch wheels).

## Architecture
- Startup: `webui.sh` creates `venv/` → `launch.py` → `modules/launch_utils.py:prepare_environment()` (pip-installs torch and `requirements_versions.txt`, clones pinned repos into `repositories/`) → `webui.py` (`webui()`, or `api_only()` with `--nowebui`).
- `modules/` is a flat package with one file per concern:
  - `processing.py`: the `StableDiffusionProcessingTxt2Img/Img2Img` classes and `process_images()`, which is the generation loop. `txt2img.py`/`img2img.py` turn UI args into these objects.
  - `postprocessing.py` runs the Extras tab. Upscale and face restore are themselves `ScriptPostprocessing` classes in `scripts/postprocessing_*.py`.
  - `shared.py` holds the global singletons (`shared.opts`, `shared.state`, `shared.sd_model`). CLI flags live in `cmd_args.py` and are read as `shared.cmd_opts`.
  - `shared_options.py` holds every setting. Settings are rendered automatically in the Settings tab and exposed at `/sdapi/v1/options`.
  - `api/api.py` defines the `/sdapi/v1/*` routes. In `api/models.py`, the txt2img/img2img request models are **generated** from the processing classes' `__init__` signatures (`PydanticModelGenerator`), so a new processing field shows up in the API automatically.
  - `sd_hijack*.py`, `sd_samplers*.py`, `sd_models*.py`: monkey-patches over the ldm/sgm code in `repositories/`, plus samplers and checkpoint loading.
  - `scripts.py` defines the `Script` base class (`title/show/ui/run`). `show()` returning `scripts.AlwaysVisible` means the script is an accordion that runs on every generation. `script_callbacks.py` has the `on_*` hooks.
- `scripts/`: built-in entries in the Script dropdown (xyz_grid, loopback, …). `extensions-builtin/` holds first-party extensions (Lora, LDSR, SwinIR, soft-inpainting…) in the same layout third-party ones use: `scripts/*.py`, optional `preload.py` (adds CLI args), `javascript/`, `style.css`.
- Frontend: `script.js` + `javascript/*.js` are plain browser scripts sharing globals, with no bundler or ES modules. `html/` holds the extra-networks card templates.

## Commands
```bash
# setup without a GPU or model (mirrors CI)
python3.10 -m venv venv && source venv/bin/activate      # python3.11 if 3.10 is absent
pip install wait-for-it -r requirements-test.txt
export STABLE_DIFFUSION_REPO=https://github.com/w-e-w/stablediffusion.git   # pinned Stability-AI repo was deleted; mirror has the same commit
python launch.py --skip-torch-cuda-test --exit           # installs torch + deps, clones repositories/; slow on first run

# tests are HTTP tests against a live server on :7860, so start one first
python launch.py --skip-prepare-environment --skip-torch-cuda-test --test-server --do-not-download-clip \
  --no-half --disable-opt-split-attention --use-cpu all --api-server-stop > test/stdout.txt 2> test/stderr.txt &
wait-for-it --service 127.0.0.1:7860 -t 300
python -m pytest -vv test                                # or: python -m pytest test/test_extras.py -k <name>
curl -XPOST http://127.0.0.1:7860/sdapi/v1/server-stop

# lint (exactly what CI runs)
pip install ruff==0.3.3 && ruff check .
npm i && npm run lint                                    # eslint over javascript/ + script.js; `npm run fix` autofixes
```
`--test-server` loads `test/test_files/empty.pt` as the checkpoint, so no SD model download is needed. Its images are noise, which is why tests assert status codes, not pixels. `test_face_restorers.py` does download GFPGAN/CodeFormer weights on first run. On python.org builds of Python, set `export SSL_CERT_FILE="$(python -m certifi)"` first, or those tests fail with `CERTIFICATE_VERIFY_FAILED`. `test/stdout.txt`/`stderr.txt` are gitignored. For a normal run, use `./webui.sh`, with persistent flags set in `COMMANDLINE_ARGS` in `webui-user.sh`.

## Conventions
- New setting: add `"key": OptionInfo(default, "Label", gr.Slider, {...}).info("hint")` to the matching `options_section(...)` block in `shared_options.py` (`.needs_reload_ui()` if it changes the layout), and read it as `shared.opts.key`. Prefer a setting over a new CLI flag.
- Reproducible generation params go into `p.extra_generation_params["Name"]`, which writes them into the PNG infotext. A script that wants "send to txt2img" to restore a param also fills `self.infotext_fields`.
- Imports: `from modules import shared, images, errors`. Imports placed inside functions are deliberate (startup time, circular imports), so don't hoist them.
- Report non-fatal failures with `errors.report("msg", exc_info=True)`. One extension's or script's exception must not abort generation.
- Python has no formatter. Don't reformat lines you didn't change. Long lines are fine (E501 is ignored).
- JS uses 4-space indent, semicolons, `function foo()` with no space before the paren, and `{a: 1}` with no brace padding. Reach the DOM through `gradioApp().querySelector(...)` and hook in with `onUiLoaded`/`onUiUpdate`/`onOptionsChanged`. **Any new cross-file global must be added to `globals` in `.eslintrc.js`**, or lint fails.
- Tests are plain pytest functions that call endpoints with `requests` through the `base_url` fixture (pytest-base-url). Image fixtures come from `test/conftest.py`. Keep requests tiny (64×64, `steps` ≤ 3). Put tests in `test/test_<area>.py`.
- UI label strings are localization keys (localizations match on exact text), so use plain English literals, not f-strings.

## Guardrails
- Never edit `repositories/` (launch re-checks it out at pinned commits; patch through `sd_hijack*` instead), `venv/`, `models/`, `outputs/`, or `extensions/` (user-installed third-party code that lint skips). Never edit or commit the user state in `config.json`, `ui-config.json`, `styles.csv`, or `cache.json`.
- `webui-user.sh`/`.bat` are tracked templates. Put local flags there, but don't commit flag changes.
- `modules/models/sd3/` and `modules/models/diffusion/` are ported upstream model code. Leave them alone unless the task is about them.
- Pins live in `requirements_versions.txt`. `requirements.txt` is unpinned, so `pip install -r requirements.txt` gives a different env than launch.
- Use ruff **0.3.3**: the latest ruff reports ~660 pre-existing errors on this repo. `ruff .` (no `check`) is deprecated.
- `pytest test` with no server running fails every test with connection errors, which is not a code bug. Start the server through `launch.py` (not `python webui.py`) so `--test-server` setup applies. Add `--skip-torch-cuda-test` on Mac/CPU, or launch aborts.
- `master` is release-only and CI fails PRs that target it. Feature work branches from and targets `dev`.
