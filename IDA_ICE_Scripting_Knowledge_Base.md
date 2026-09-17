# IDA ICE SCRIPTING KNOWLEDGE BASE
Version 1.0 — reference for the "IDA ICE Automation Engineer" Copilot agent
Sources: EQUA "Learn some IDA scripting" (Power User Days 2015, summarised), EQUA forum answers, and tested in-house scripts.
Language: IDA ICE script language (Lisp-based). Comments start with ";".

## CONTENTS
1. Where and how scripts run (execution contexts)
2. Syntax fundamentals
3. References to objects, parameters and attributes
4. Keyword navigation functions (:zones, :walls, :windows ...)
5. Core script commands
6. Useful functions (geometry, selection, results, resources)
7. Pattern library — annotated working scripts
8. Macro injection pattern (monitoring / KPI macros)
9. Custom script object with inputs (parametric runs, MOBO)
10. Command-line and batch execution
11. Known object, parameter and variable names
12. Forum knowledge (Q&A insights)
13. Pitfalls and debugging checklist

--------------------------------------------------------------------------------
## 1. WHERE AND HOW SCRIPTS RUN (EXECUTION CONTEXTS)

A. Tools > Run script (general window)
- Opens an "Execute script" dialog with a ROOT field and a CODE field.
- The root is normally the building. Inside the script [@] refers to the root.
- Best place to develop: scripts can be tested incrementally.
- Good for: whole-building loops, loading resources, running simulations, bulk edits.

B. Run script on selected objects (Outline / tables)
- Select several objects in the Outline or a table (e.g. several surfaces), right-click, "Run script on...", paste code.
- The script runs once per selected object; [@] is the CURRENT selected object.
- Pattern: (:FOR (S [@]) ...) or directly (:UPDATE [@] ...).
- Good for: adding windows to many surfaces, editing selected zones.

C. Script operating on 3D selection
- (:SET PANE_ (:CALL FIND-VIEW-PANE [@] THREE-D T)) gets the 3D view.
- (:CALL SELECTED-DATA-OBJECTS PANE_) returns objects selected in 3D (shift-click to multi-select).

D. Custom script object inside a system/macro (inputs and outputs)
- A script placed in the model whose input boxes (left side) are read as [|u_var| 1], [|u_var| 2] ...
- Inputs can be connected to Parametric runs or MOBO optimisation, so the script recomputes parameters for each variant (see section 9).

E. Diff script in version projects
- IDA creates a diff script automatically when comparing two versions.
- Often easier to write manually: make a change in one zone, save as version, then edit the diff script to loop over all zones.
- Tools > DIFF-SCRIPT shows all changes made to the open model (useful to "record" a GUI action and learn its script syntax).

F. Command line (see section 10).

HOW TO FIND REAL NAMES
- Options/Preferences/Developer: enable "Do not translate names". The Outline then shows internal names usable in scripts.
- Objects can be copied from the Outline and pasted (or dragged) into the script code field — this reveals the exact syntax of any object, including macros.

--------------------------------------------------------------------------------
## 2. SYNTAX FUNDAMENTALS

- Every command is a list in parentheses: (command_name arg1 arg2 ...). Arguments are separated by blanks or newlines.
- Arguments can be literal values (numbers, "strings", symbols), script variables, references [ ... ], or nested expressions.
- Case-insensitive for symbols: :FOR and :for are equivalent.
- Local variable names conventionally end with "_" (zone_, area_) to avoid clashes with object names.
- Strings use straight double quotes "..." — never typographic quotes (“ ”) copied from web pages or Word.
- Quoting: 'X or (QUOTE X) prevents evaluation (used for lists of model types and symbols, e.g. '(IDEAL-HEATER IDEAL-COOLER), 'det-window).
- Booleans in parameters: :TRUE / :FALSE. In Lisp tests: T / NIL.
- Arithmetic is prefix: (+ a b), (- a b), (* a b), (/ a b). Comparisons: (= a b) numeric, (== a b) general equality, <, >, <=, >=. Range test: (<= 145 x 180).
- Logic: (:AND ...), (:OR ...) or (and ...), (or ...), (not ...).
- Units are SI: m, W, °C, kg/s, s (1 h = 3600 s), m3/s. Mass flow in kg/s (multiply by 3600 for kg/h).
- Both ":call"-prefixed and plain forms appear in EQUA examples: (:call delete-component x) and (delete-component x); (set-value ...). Prefer the :CALL form for functions when unsure.

--------------------------------------------------------------------------------
## 3. REFERENCES TO OBJECTS, PARAMETERS AND ATTRIBUTES

- [@]                    the base (root) object
- [@ name1 name2 ...]    a (sub)component object by path from the base
- [name1 name2 ...]      the VALUE of a parameter/variable (same as [@ ...] if the target is not a parameter/variable)
- [name1 ... attribute]  an attribute of an object
- [@ var_ name1 ...]     path starting from an object stored in a variable
- [var_ name1 ...]       value starting from a variable, e.g. [zone_ geometry net_floor_area], [zone_ name], [Z_ GROUP]
- (step1 step2 ...)      unevaluated path, passed to functions that treat it as a reference, e.g. (set-value zone_ ("Occupant" activity_level) 0.6)
- [@ :system "Zone1"]    find a named zone within the system

Path steps may be:
- a component name (symbol or "string") or attribute name
- a keyword function (starts with ":"), applied to the previous object, e.g. [@ :zones]
- an integer index (lists start at 1, vectors start at 0)
- * for an array slice
- an expression evaluated to the target object (quote it with ' to avoid double evaluation)
- For resource parameters, a step that doesn't fit the parameter is applied to the referenced resource.
- For a list of objects: a name step returns that member (or is applied to every member); other steps are applied to every member and return a list.

Nested building reference used inside other scopes: [[@ :BUILDING] :ZONES]

--------------------------------------------------------------------------------
## 4. KEYWORD NAVIGATION FUNCTIONS

General (all IDA applications):
:parent (containing object), :model (smallest NMF/Modelica model, boundary, output or macro containing it), :macro, :system, :root, :simres (result collection of an output object), :lib, :sys, :self, :ref (referenced resource), :name, :full-name, :global-name, :value, :actual-value (resolves :default), :unit, :total (sum of a list value), :description, :type, :given (variable binding text), :parameters, :variables, :interfaces, :resources, :components, :boundaries, :results, :versions, :all-variables, :all-parameters, :all-resources, :all-results, :origin (physical object of a math model), :logged-to (output file a variable is logged to), :array-elements, :error.

IDA ICE specific:
- :building — the building object
- :section — building body containing the object
- :zone — zone containing the object; :zone-ex — same, but falls back to the zone template [@ :building defaults zones]
- :wall — zone wall/ceiling/floor containing the object
- :zones — zones in building or result collection
- :control-macros, :energy-meters, :hvac-components (AHUs and plants), :ice-schedules
- :ice-all-enclosing — all zone walls, floors and ceilings in building or zone
- :loads (equipment, occupants, lights), :equipments, :occupants, :lights, :masses
- :clothing — active clothing parameter
- :plant, :ahu — the plant/AHU macro containing the object; :central-ahu — zone's main AHU; :ahus — AHU clients and local AHUs in zone; :all-ahus
- :faces — building-body walls, floors, roofs (complex roof as one object when used as a step)
- :surfaces — zone walls, floors, ceilings; :walls — zone walls only
- :openings, :windows — belonging to the object
- :features — objects inserted into zone surfaces belonging to the object
- :ice-measuring-planes — measuring planes (used as [[@ :BUILDING] :ice-measuring-planes])

--------------------------------------------------------------------------------
## 5. CORE SCRIPT COMMANDS

LOOPS
(:FOR (var_ list_expr [:WHEN test] [:UNLESS test] [:SAVE (vars)]) body...)
- :WHEN / :UNLESS filter items.
- :SAVE (v1 v2) makes variables modified inside the loop keep their values outside it.
- Numeric ranges: (:FOR (i (:RANGE 1 N)) ...)
- Literal lists: (:for (z_ ("Zone1" "Zone2")) ...)

VARIABLES
(:SET var_ value)                      single
(:SET a_ 1 b_ 2)                       several at once
(:SET (VAR value))                     alternative form seen in examples
(:SET ((l_ b_ r_ t_) list_expr))       destructuring a 4-element list

CONDITIONALS
(:COND (test1 body1...) (test2 body2...))
(? expr) — true if expr returns something (used to test existence, e.g. (? (:CALL COMPONENT ZONE_ "Hcrad_1")))

MODIFYING OBJECTS
(:UPDATE target
   (:PAR :N param_name :V value)                  set parameter
   (:VAR :N var_name :B binding :L "logfile" :AS "alias")   set variable binding/logging
   (:RES :N resource_param :V "Resource name")    set resource parameter
   (:REMOVE "component name")                     remove component
   (:ADD (TYPE :N "name" :T template :D "description") sub-updates...)   add component
   ((TYPE :N existing_name) sub-updates...)       descend into existing component
   ((:EO :N "name") ...)                          descend into equation object by name
)
- :N name, :T type/template, :D description.
- :PAR option :S '(:DEFAULT NIL 2) appears in copied objects (display state); can be kept or omitted.
- :DIM (n) sets array dimension; (:PAR :N (X 3) :V 1.5) sets array element 3.

(set-value object path value)          e.g. (set-value [@ output] 'heat_balance :True)
(replace-model object new_type_or_template [:reset-values :none])
(:call make-component parent ((type :t template :n "name") (:par ...)))   — with parent NIL creates a temporary object
(:call delete-component object)
(load-resource-from-db [@] 'resource_class "Resource name")   classes seen: 'walldef, 'windef

BATCH / ERROR HANDLING / OUTPUT
(:BATCH > NIL commands...)             run in batch mode; "> NIL" suppresses per-iteration log output
(:set _ERRORLEVEL 4)                   raise error-level threshold so warnings don't stop a loop
(:set out_ (:call open-output "C:\\path\\name.txt"))
(format out_ "Location: ~A~%" [location])   Lisp FORMAT: ~A value, ~% newline
(:clean (close out_))                  ensures the file closes even on error

SIMULATIONS
(:call ice-run-load-sim-ex [@ simulations heating])
(:call ice-run-load-sim-ex [@ simulations cooling])
(:call find-sim-results [@] cooling)   results of a given simulation (since ICE 4.7 results are stored per simulation)

--------------------------------------------------------------------------------
## 6. USEFUL FUNCTIONS (as used in working scripts)

Geometry / surfaces
- (:call wall-extent-sides S) → nested list of left, bottom, right, top of a surface; use with flatten:
  (:set ((left_ bottom_ right_ top_) (:call flatten (:call wall-extent-sides S))))
- (:call wall-size-x w), (:call wall-size-y w) → width/height of a wall or floor (width is not a parameter because walls can have custom shapes)
- (:call wall-azimut w) → azimuth of a wall (note the spelling "azimut")
- (:call wall-external-part w) → fraction of wall area that is external (e.g. > 0.9 to select external walls)
- [w geometry slope] → 90 = vertical wall, 180 = flat roof side, >= 90 selects walls and roofs
- (:call zone-ceiling zone_) → the ceiling of a zone
- (:call ice-zone-enclosing-surfaces zone_ t) → external surface parts of a zone
- [zone_ geometry net_floor_area], [zone_ geometry floor_height_from_ground]
- (:call ICE-UPDATE-MEASURING-PLANE-SHAPE p) → refresh measuring plane geometry

Selection / lookup
- (:call component zone_ "Name") → component by name (NIL if missing)
- (:call list-models '(IDEAL-HEATER IDEAL-COOLER) zone_) → models of given types in zone
- (:call list-features zone_ floor-heat) → inserted features of a type
- (:call model-type obj) → type symbol, compare with (== ... 'det-window)
- [obj type] → template/type name string, e.g. (== [win_ type] "YY")
- (:call string-equal "Daylight" [Z_ GROUP]) → case-insensitive string compare (zone group filter)
- (:call equal a b), (:call nth 0 list), (:call cons x list), (:call flatten list)
- (:call find-view-pane [@] three-d t), (:call selected-data-objects pane_)

Results
- [cool_ zn_ cooling-summary roomunitcool], [heat_ zn_ heating-summary roomunitheat]
- (:CALL NTH 0 ([[@ :BUILDING] ENERGY-REPORT DEMAND "Fuel heating"])) → peak/demand value from energy report

--------------------------------------------------------------------------------
## 7. PATTERN LIBRARY — ANNOTATED WORKING SCRIPTS

### 7.1 Loop over zones filtered by group, set values from floor area
Context: Tools > Run script, root = building.
(:FOR (Zeta [@ :ZONES] :WHEN (:call string-equal "typ2" [Zeta group]))
  (:UPDATE Zeta
    ((SETPOINT_COLLECTION :N LOCALSETPOINTS)
      (:PAR :N MIN_VENT_AIR :V (/ 60 [Zeta GEOMETRY NET_FLOOR_AREA]))
      (:PAR :N MAX_VENT_AIR :V (/ 70 [Zeta GEOMETRY NET_FLOOR_AREA]))
      (:PAR :N MIN_VENT_SUP :V (/ 80 [Zeta GEOMETRY NET_FLOOR_AREA]))
      (:PAR :N MAX_VENT_SUP :V (/ 90 [Zeta GEOMETRY NET_FLOOR_AREA])))))
Note: the component name in the original forum post appeared both as LOCAL_SETPOINTS and LOCALSETPOINTS — verify in Outline.

### 7.2 Filter on several groups at once
(:for (zn_ [[@ :BUILDING] :zones]
       :when (:or (:call string-equal "Panel" [zn_ group])
                  (:call string-equal "Fan" [zn_ group])
                  (:call string-equal "General" [zn_ group])))
  (:UPDATE zn_ ...))

### 7.3 Set value in all zones except one / per-area values
(:for (zone_ [@ :zones] :when (not (== "No change" [zone_ name])))
  (set-value zone_ ("Occupant" activity_level) 0.6))

(:for (zone_ (:call :zones [@]) :unless (:call equal "No change" [zone_ name]))
  (:set area_ [zone_ geometry net_floor_area])
  (set-value zone_ (dhw hot-water-use) (/ 10 area_)))

### 7.4 Delete zones above a height
(:for (zone (:call :zones [@]) :when (> [zone geometry floor_height_from_ground] 2))
  (:call delete-component zone))

### 7.5 Remove all ideal heaters and coolers
(:FOR (ZONE_ (:CALL :ZONES [@]))
  (:FOR (UNIT_ (:CALL LIST-MODELS '(IDEAL-HEATER IDEAL-COOLER) ZONE_))
    (:CALL DELETE-COMPONENT UNIT_)))
Coolers only: use '(IDEAL-COOLER).

### 7.6 Size cooling beams from design simulation results
(:for (zone_ [@ :zones])
  (:for (unit_ (:call list-models '(ideal-heater ideal-cooler) zone_))
    (:call delete-component unit_)))
(:set cool_ (:call find-sim-results [@] cooling)
      heat_ (:call find-sim-results [@] heating))
(:for (zone_ [@ :zones])
  (:set zn_ [zone_ name])
  (:set cool_pw_ [cool_ zn_ cooling-summary roomunitcool]
        heat_pw_ [heat_ zn_ heating-summary roomunitheat])
  (:cond ((and (< 0 cool_pw_) (< 0 heat_pw_))
    (:update zone_
      ((therm_beam :n "beam" :t therm_beam)
        (:par :n cool_power :v cool_pw_)
        (:par :n cool_power_0 :v (* 0.1 cool_pw_))
        (:par :n air_flow_total :v (* (/ cool_pw_ 500) 20))
        (:par :n heat_power :v heat_pw_)
        (:par :n heat_power_0 :v (* 0.1 heat_pw_)))))))
Prerequisite: run the heating and cooling design simulations first (7.12) while the ideal units still exist; the stored results remain readable after the units are deleted.

### 7.7 Replace floor heating/cooling with ideal units of equal power
(:for (zone [@ :zones])
  (:set cool 0 heat 0)
  (:for (hc (:call list-features zone floor-heat)
         :when (== (:call model-type hc) 'therm_floor)
         :save (cool heat))
    (:set cool (+ cool [hc pcool])
          heat (+ heat [hc pheat]))
    (delete-component hc))
  (:cond ((> cool 0)
    (make-component zone
      ((he-unit :t ideal-cooler :n "Floor cooler replacement")
        (:par :n pmax :v (* cool [zone geometry net_floor_area]))))))
  (:cond ((> heat 0)
    (make-component zone
      ((he-unit :t ideal-heater :n "Floor heater replacement")
        (:par :n pmax :v (* heat [zone geometry net_floor_area])))))))
Note: PCOOL/PHEAT are specific (W/m2), hence multiplication by floor area.

### 7.8 Add floor heating/cooling at 80 % of floor area in named zones
(:for (z_ ("Zone1" "Zone2"))
  (:update [@ :system z_]
    (:REMOVE "Ideal cooler")
    (:REMOVE "Ideal heater")
    ((ENCLOSING-ELEMENT :N FLOOR)
      (:ADD (FLOOR-HEAT :N "hc-floor" :T THERM_FLOOR :D "Heating/cooling floor")
        (:PAR :N DX :V (* 0.8 (:call wall-size-x [z_ floor])))
        (:PAR :N DY :V (* 0.8 (:call wall-size-y [z_ floor])))))))
For WALL heating: same FLOOR-HEAT element inserted into a wall (ICE 4.8+). Difficulty: choosing which wall — filter by (:call wall-external-part w), azimuth or slope.

### 7.9 Add windows to many selected surfaces (percentage sizing, centred)
Context: select surfaces in Outline > right-click > Run script on...
(:BATCH > NIL
  (:SET (VERTICAL-PERCENT 0.8))
  (:SET (SIDE-OFFSET-PERCENT 0.8))
  (:FOR (S [@])
    (:set ((left_ bottom_ right_ top_) (:call flatten (:call wall-extent-sides S))))
    (:UPDATE S
      (:ADD (CE-WINDOW :N "FE-Dome" :T "FE-Dome")
        (:PAR :N X  :V (+ left_ (* (/ (- 1 SIDE-OFFSET-PERCENT) 2) (- right_ left_))))
        (:PAR :N Y  :V (- top_ (* VERTICAL-PERCENT (- top_ bottom_))))
        (:PAR :N DX :V (* SIDE-OFFSET-PERCENT (- right_ left_)))
        (:PAR :N DY :V (* VERTICAL-PERCENT (- top_ bottom_)))))))
- X, Y = lower-left corner of the window in surface coordinates; DX, DY = width, height.
- Window hangs from the top edge (Y measured down from top_). For a sill height h: Y = bottom_ + h.
- :T must be an existing window template/resource in the model ("FE-Dome" here).
- Variant: generic windows on walls from a zone loop:
(:for (zone_ (:call :zones [@]))
  (:for (w (:call ice-zone-enclosing-surfaces zone_ t)
         :when (and (>= [w geometry slope] 90) (> (:call wall-external-part w) 0.9)))
    (:update w (:ADD (CE-WINDOW :T "FT1") (:PAR :N DX :V 1)))))

### 7.10 Lighting in every zone scaled to floor area, covering the ceiling
; Assumes the lighting resource holds specific load W/m2; NUMBER_OF = floor area
(:for (zone_ [@ :zones])
  (:SET ZF_ [zone_ GEOMETRY NET_FLOOR_AREA])
  (:set c_ (:call zone-ceiling zone_))
  (:set ((left_ bottom_ right_ top_) (:call flatten (:call wall-extent-sides c_))))
  (:UPDATE zone_
    (:ADD (LIGHT :N "Light" :T "Büro Licht")
      (:PAR :N NUMBER_OF :V ZF_)
      (:par :n x :v left_)
      (:par :n y :v bottom_)
      (:par :n dx :v (- right_ left_))
      (:par :n dy :v (- top_ bottom_)))))
Customise :T to the lighting resource name in the model.
(The original script also contains a redundant line (:SET ZF_ [ZN_ GEOMETRY ...]) via the name; using zone_ directly is cleaner.)

### 7.11 Measuring planes in zones of group "Daylight"
(:FOR (Z_ [[@ :BUILDING] :ZONES] :WHEN (:CALL STRING-EQUAL "Daylight" [Z_ GROUP]))
  (:UPDATE Z_
    ((ENCLOSING-ELEMENT :N FLOOR)
      (:ADD (MEASURING-PLANE :N "Plane" :T MEASURING-PLANE :D "Daylight measuring plane")
        (:PAR :N OFFSET :V 0.1)
        (:PAR :N USE-OFFSET :V :TRUE)))))
(:for (p [[@ :BUILDING] :ice-measuring-planes])
  (:call ICE-UPDATE-MEASURING-PLANE-SHAPE p))
To apply to all zones remove the :WHEN clause (keep bracket balance).

### 7.12 Enable heat balance and run design simulations
(set-value [@ output] 'heat_balance :True)
(:call ice-run-load-sim-ex [@ simulations heating])
(:call ice-run-load-sim-ex [@ simulations cooling])

### 7.13 Change external construction of south walls and shallow roofs
(load-resource-from-db [@] 'walldef "Frame wall 195mm (example)")
(:set out nil)
(:for (zone_ (:call :zones [@]) :save (out))
  (:for (w (:call ice-zone-enclosing-surfaces zone_ t) :save (out)
         :when (>= [w geometry slope] 90))
    (:set out (:call cons w out))))
(:for (w out :when (or (<= 145 [w geometry slope] 180)
                       (> 200 (:call wall-azimut w) 160)))
  (set-value w 'construction_external "Frame wall 195mm (example)"))
Pattern: collect objects in a list with :save + cons, then act on the list.

### 7.14 Replace selected (3D) windows by detailed windows with a glazing
(load-resource-from-db [@] 'windef "Double Clear Air (WIN7)")
(:set tmp_win_ (:call make-component nil
  ((nil :t det-window)
   (:res :n glazing :v "Double Clear Air (WIN7)"))))
(:set pane_ (:call find-view-pane [@] three-d t))
(:for (win_ (:call selected-data-objects pane_) :when (== (:call model-type win_) 'window))
  (replace-model win_ tmp_win_))
Why the temp object: replacing directly with DET-WINDOW leaves glazing undefined.

### 7.15 Convert ALL windows standard <-> detailed
(:set _ERRORLEVEL 4)
(:for (win_ [@ :windows] :when (== (:call model-type win_) 'window))
  (replace-model win_ 'det-window :reset-values :none))
Back to simple:
(:for (win_ [@ :windows] :when (== (:call model-type win_) 'det-window))
  (replace-model win_ 'window))

### 7.16 Set a double-glass-facade parameter from window area (by window type)
(:for (win_ [@ :windows] :when (== [win_ type] "YY"))
  (:set warea (* [win_ dx] [win_ dy]))
  (:update win_
    ((AGGREGATE :N DOUBLE-GLASS_FACADE)
      (:PAR :N AMB_DOF_FLOOR :V (/ warea 8.7))
      (:PAR :N AMB_DOF_CEILING :V (/ warea 8.7)))))

### 7.17 Boiler size from energy report peak demand
(:SET POWER (:CALL NTH 0 ([[@ :BUILDING] ENERGY-REPORT DEMAND "Fuel heating"])))
(:UPDATE [@ :BUILDING]
  ((PRIM-MACRO :N PLANT)
    ((:EO :N "boil")
      (:PAR :N QMAX :V POWER))))
Tested in IDA ICE 4.8. Peak power comes from the energy meter linked to the boiler (includes primary energy factor/efficiency if defined).

### 7.18 Replace plant or AHU wholesale
(REPLACE-MODEL [@ PLANT] ESBO-PLANT)
(REPLACE-MODEL [@ AHU] "lib:ahu/ASHRAE_toolkit.idm")

### 7.19 Conditional insertion only where a component exists
(:FOR (ZONE_ [[@ :BUILDING] :ZONES])
  (:COND ((? (:CALL COMPONENT ZONE_ "Hcrad_1"))
    (:UPDATE ZONE_ (:ADD ...)))))

### 7.20 Write custom text output
(:set out_ (:call open-output "C:\\Temp\\zones.txt"))
(:for (zone_ [@ :zones])
  (format out_ "~A;~A~%" [zone_ name] [zone_ geometry net_floor_area]))
(:clean (close out_))

--------------------------------------------------------------------------------
## 8. MACRO INJECTION PATTERN (MONITORING / KPI MACROS)

Goal: add the same monitoring macro (references + math + logging) into many zones.

Workflow (recommended — do not hand-write graphics):
1. Build the macro once in one zone via the GUI (Advanced level): references to zone variables, adders/multipliers/sliding averages, output file.
2. Copy the macro object from the Outline and paste it into the script window — IDA writes the full definition.
3. Wrap it in a zone loop:
(:for (zn_ [[@ :BUILDING] :zones] :when <filter>)
  (:UPDATE zn_
    (:ADD (MACRO-OBJECT SCHEMA '(...pasted drawing data...) :SYMBOL '(...) :N "Monitor_CC" :T ICE-MACRO :D "ICE macro")
      ...pasted components...
      (CONNECTIONS ...pasted...))))
4. Optionally guard with (:COND ((? (:CALL COMPONENT zn_ "Hcrad_1")) ...)) so it only goes where the referenced component exists — otherwise references break.

Anatomy of pasted macro content:
- SCHEMA / :SYMBOL '(:AT ((x y)) ...) — graphical layout only (positions, icons). Keep as pasted.
- OUTPUT-FILE ... :SF "self:\\Monitor_CC.prn" :N "Monitor_CC" :T OUTPUT-FILE :COL T [:RP T] — log file in the zone results.
- REFERENCE element reading a zone variable:
  ((REFERENCE :SYMBOL '(...) :N "REFERENCE1" :T REFERENCE :D "Supply water temperature")
    (:REPLACE CONNECTOR :N LINK :T |OutPort| :F 32 :V '((:ZONE "Hcrad_1" TIN))))
  Path forms: (:ZONE NMFZONE TAIRMEAN), (:ZONE "Hcrad_1" TSURF), (:BUILDING CLIMATE TAIR2), (:MACRO "IntStatM" |uMax_var|)
- Math element logging its result:
  ((:EO :N "ADDER1" :T ADDER)
    (:PAR :N N_IN :V 1) (:PAR :N COEFF :DIM (1))
    (:VAR :N INSIGNAL :DIM (1) :B #S(MS-SPARSE ... ((1 -1 (INSIGNALLINK 1) 0))) :L "Monitor_CC" :AS "TIN"))
  :L = output file name to log to, :AS = column alias.
- Modelica-type blocks use |pipes|: ((MODEL :N "SlideAvg" :T |SlidingAverage|) (:VAR :N |u_var| ...) (:VAR :N |uMean_var| :L "file" :AS "alias") (:PAR :N |interval| :V 3600))
- CONNECTIONS list: (("TARGET" '(INSIGNALLINK 1)) ("SOURCE" LINK) 0 0 '(:AT (...points...) ...))

Compact alternative (no references, no drawing): add blocks with direct bindings to zone variables:
(:FOR (ZONE_ [[@ :BUILDING] :ZONES])
  (:COND ((? (:CALL COMPONENT ZONE_ "Hcrad_1"))
    (:UPDATE ZONE_
      (:ADD (MODEL :N "Feedback" :T |Feedback|)
        (:VAR :N |u1_var| :B (1 NMFZONE DEWP))
        (:VAR :N |u2_var| :B (1 "Hcrad_1" TIN)))
      (:ADD (MODEL :N "SlideAvg" :T |SlidingAverage|)
        (:VAR :N |u_var| :B #S(MS-SPARSE DEFAULT-VALUE NIL DIMENSION 1 VALUE ((1 1 "Feedback" |y_var|))))
        (:PAR :N |interval| :V 900))
      (:ADD (:EO :N "MINMAXD" :T MINMAXD)
        (:PAR :N SELECTOR :V 1)
        (:VAR :N INSIGNAL :B #S(MS-SPARSE DEFAULT-VALUE NIL DIMENSION 1 VALUE ((1 1 "SlideAvg" |uMean_var|) (2 . 0.0)))))
      (:ADD (MODEL :N "GreaterThan" :T |GreaterThan|)
        (:PAR :N |threshold| :V 0.01)
        (:VAR :N |u_var| :B (1 "MINMAXD" OUTSIGNAL)))
      (:ADD (MODEL :N "DP_DUR" :T |Integrator|)
        (:PAR :N |k| :V 2.777778E-4)
        (:VAR :N |u_var| :B (1 "GreaterThan" |y_var|)))
      (:ADD (MODEL :N "DP_DEGHR" :T |Integrator|)
        (:PAR :N |k| :V 2.777778E-4)
        (:VAR :N |u_var| :B (1 "MINMAXD" OUTSIGNAL)))))))
This KPI chain computes dew-point risk: difference dew point − supply temp (15-min average), clipped at 0, counts hours above 0.01 K (DP_DUR, k = 1/3600 converts s→h) and degree-hours (DP_DEGHR).
(The original also carries :SYMBOL layout data for each block; omitting it may place blocks at default positions — test.)

Binding syntax summary:
- (1 NMFZONE DEWP) — bind to variable DEWP of zone model NMFZONE
- (1 "Name" VAR) — bind to variable of a sibling component
- (-1 |u| 0) — bind to own interface port
- #S(MS-SPARSE DEFAULT-VALUE NIL DIMENSION 1 VALUE ((i ...) ...)) — array binding; (2 . 0.0) = constant 0 on element 2

Common signal blocks seen: ADDER (N_IN, COEFF), MULTIPLIER, |Division|, SOURCE-CONSTANT (param X), |SlidingAverage| (interval s), |IntStatM| (interval; uMean/uMin/uMax), |SnapMinMax| (threshold; uMin/uMax annual), |Duration| (level vector, threshold; tu = time above level, s), |Feedback| (u1 − u2 → y), MINMAXD (SELECTOR 1 = max), |GreaterThan| (threshold → y 0/1), |Integrator| (k gain).

Typical monitoring macros in the in-house library:
- Monitor_CC: concrete-core / radiant (Hcrad_1) P, TIN, TOUT, TSURF, mass flow kg/h, zone dew point, local absolute humidity.
- results (sliding averages 1 h): PMV, TAIRMEAN, TOP, SCHEDOCC, outdoor TAIR2, RELATIVEHUM, XHUM.
- Schwankung: 24 h mean/min/max (IntStatM, interval 86400), annual min/max (SnapMinMax), fluctuation max−min, exceedance time above 22 °C (Duration).
- OutpZn general: zone heat balance terms (EMETER references such as EMETERQ_EXTWIND OUTPOWERW, EMETERQ_INTSURF POSPOWER) divided by floor area and averaged hourly.

--------------------------------------------------------------------------------
## 9. CUSTOM SCRIPT OBJECT WITH INPUTS (PARAMETRIC RUNS / MOBO)

Inputs are read as [|u_var| i]. Example: borehole field layout computed from 4 inputs.
; Inputs: 1 = boreholes in X, 2 = boreholes in Y, 3 = spacing X [m], 4 = spacing Y [m]
(:UPDATE [@ :BUILDING]
  ((PRIM-MACRO :N PLANT)
    ((MACRO-OBJECT :N "EWS_CC")
      ((:EO :N "Ghx_Mir")
        (:PAR :N NHOLE :V [|u_var| 5])
        (:PAR :N (NG 1) :V [|u_var| 5])
        (:SET (NXX [|u_var| 1]))
        (:SET (NYY [|u_var| 2]))
        (:SET (DXX [|u_var| 3]))
        (:SET (DYY [|u_var| 4]))
        (:FOR (NYI (:RANGE 1 NYY))
          (:FOR (NXI (:RANGE 1 NXX))
            (:SET (NI (+ NXI (* NXX (- NYI 1)))))
            (:PAR :N (X NI) :V (+ (/ DXX 2) (* DXX (- NXI 1))))
            (:PAR :N (Y NI) :V (+ (/ DYY 2) (* DYY (- NYI 1))))))))))
Notes:
- Original script uses 4 :COND branches for first row/column; they all reduce to the general formula above (X = DXX/2 + DXX·(i−1)), so the compact version is equivalent.
- The original header mentions 4 inputs but reads [|u_var| 5] for number of holes: the object needs 5 inputs (5 = total boreholes = NXX·NYY). Alternatively compute NHOLE as (* NXX NYY).
- Array parameters are addressed as (X NI), (Y NI), (NG 1).
- Connect input boxes to parametric-run or MOBO variables so geometry updates for each variant.

--------------------------------------------------------------------------------
## 10. COMMAND-LINE AND BATCH EXECUTION

- path\ice.exe arguments — starts ida-ice.exe if needed, passes arguments, returns immediately.
- path\ida-ice.exe path\ida.img arguments
- Multiple instances: add -C id (unique id each) so temporary folders don't collide.
- Run a script: path\ice.exe [-C id] [-B [logfile]] [-Q] -X script [var value ...]
  script = a command, a list of commands in parentheses, or a "file name" in double quotes.
- Before -X: command-line syntax (blank-separated, quotes for spaces). After -X: IDA list syntax.
- -B batch mode: pop-ups go to log window/file. -Q: quit after execution (ICE 4.7 Beta 26+).
- Process files: path\ice.exe [-C id] [-Q] -E script -R file [file2 ...]
  or path\ice.exe [-C id] [-Q] -E script -B [logfile] file [file2 ...]
  script can be a simulation type (heating, cooling, energy, daylight) or a script applied to each file. @file = response file listing files.
- Useful with Python/VBA orchestration: generate script files, launch ice.exe per case with unique -C ids.

--------------------------------------------------------------------------------
## 11. KNOWN OBJECT, PARAMETER AND VARIABLE NAMES (seen in working scripts)

Zone model variables (path (:ZONE NMFZONE var) or (1 NMFZONE var)):
TAIRMEAN (well-mixed air temp), TOP (operative temp), PMV, DEWP (dew point), RELATIVEHUM, XHUM (humidity ratio kg/kg), XHUMLOC (local humidity ratio), SCHEDOCC (occupancy on/off)
Climate: (:BUILDING CLIMATE TAIR2) outdoor air temperature
Radiant/CC component "Hcrad_1": P (emitted heat W), TIN, TOUT, TSURF, M (kg/s)
Energy meters in zone: EMETERQ_EXTWIND OUTPOWERW, EMETERQ_INTSURF POSPOWER
Zone object types/components: SETPOINT_COLLECTION LOCALSETPOINTS (MIN_VENT_AIR, MAX_VENT_AIR, MIN_VENT_SUP, MAX_VENT_SUP), ENCLOSING-ELEMENT FLOOR, FLOOR-HEAT (template THERM_FLOOR; DX, DY; PCOOL, PHEAT), LIGHT (NUMBER_OF, X, Y, DX, DY), MEASURING-PLANE (OFFSET, USE-OFFSET), THERM_BEAM (COOL_POWER, COOL_POWER_0, AIR_FLOW_TOTAL, HEAT_POWER, HEAT_POWER_0), HE-UNIT with templates IDEAL-HEATER / IDEAL-COOLER (PMAX), "Occupant" (ACTIVITY_LEVEL), DHW (HOT-WATER-USE)
Surfaces/openings: CE-WINDOW (X, Y, DX, DY), WINDOW, DET-WINDOW (resource GLAZING), wall parameter CONSTRUCTION_EXTERNAL, AGGREGATE DOUBLE-GLASS_FACADE
Building level: OUTPUT (HEAT_BALANCE), SIMULATIONS (HEATING, COOLING), ENERGY-REPORT DEMAND, PRIM-MACRO PLANT, AHU, DEFAULTS ZONES
Resource classes: 'walldef, 'windef
ALWAYS verify names in the target model's Outline with "Do not translate names" — names differ between versions, templates and languages.

--------------------------------------------------------------------------------
## 12. FORUM KNOWLEDGE (Q&A INSIGHTS)

- Wall width/height are not parameters (custom shapes); use (:call wall-size-x w) / wall-size-y.
- wall-external-part is a function, not a hidden property.
- IDA ICE has no user-defined wall "types" to filter on; select walls by geometry tests (slope, azimuth, external part) or names.
- Wall heating (ICE 4.8+): insert the "Heating/cooling floor" element into a wall (drag & drop or script).
- Heating-load-simulation energy variables are energy, not peak power; for peak power use the energy report demand of the meter linked to the boiler.
- Monthly g-values of windows with external shading: log QSOLAR1 (or QSOLAR) in the window model and PDIFGRD, PDIFSKY, PDIR in ExtWall_TQFace (advanced level). g_syst = QSOLAR1 / ((PDIFGRD + PDIFSKY + PDIR) · AWINDOW · (1 − FRRATIO)). In the standard window model area/frame fraction are A and FRAMERAT. QSOLAR (instead of QSOLAR1) subtracts radiation reflected back out. Keep shading permanently on for a pure shaded value.
- Since ICE 4.7 simulation results are stored per simulation: use FIND-SIM-RESULTS.

--------------------------------------------------------------------------------
## 13. PITFALLS AND DEBUGGING CHECKLIST

1. Bracket balance: count ( ) and [ ]. Most failures are a missing ")" at the end.
2. Quotes: replace “ ” ‘ ’ with straight quotes. OCR/Word copies also turn 0 into @ or O, and ":" into "i" or "r" (e.g. ":when" → "rwhen").
3. Names: use internal (untranslated) names; :N is the instance name, :T the type/template/resource.
4. Resources used in :T or :RES must exist in the model — load with load-resource-from-db first.
5. Root: confirm what [@] is (building vs selected object). Use [[@ :BUILDING] ...] when the root may be a sub-object.
6. References in macros break if the referenced component (e.g. "Hcrad_1") is missing in a zone — guard with (? (:CALL COMPONENT ...)).
7. Duplicate names: :ADD with a name already present may fail or create "Name1"; remove first or check existence.
8. Units: SI, seconds, W, kg/s. Intervals for averages in seconds (3600 = 1 h, 86400 = 1 day).
9. Quote type lists/symbols: '(IDEAL-HEATER IDEAL-COOLER), 'det-window.
10. Loop variables modified inside nested loops need :SAVE to be visible outside.
11. Array indices: IDA lists start at 1, vectors at 0.
12. Long loops: wrap in (:BATCH > NIL ...) and consider (:set _ERRORLEVEL 4).
13. Test on a copy / a single zone first, save the model before running, check the effect in Outline/3D.
14. Learn syntax from IDA itself: copy objects from Outline into the script window, or run DIFF-SCRIPT after a manual GUI change.
15. Version differences: features/functions may vary between IDA ICE 4.7, 4.8, 5.x — state the version when unsure.
