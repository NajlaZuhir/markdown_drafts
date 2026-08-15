# Procedure Seeder Guide

## Reusable Starter Template

Reusable starter template: ProcedureSeederTemplate.php.stub

It is a `.stub` file, so Laravel will **not run it accidentally**.

1) Copy it:

```text
database/seeders/ProcedureSeederTemplate.php.stub
-> copy/rename to
database/seeders/NewProcedureSeeder.php
```
**Copy Command**

Run:

```powershell
Copy-Item database\seeders\ProcedureSeederTemplate.php.stub database\seeders\NewProcedureSeeder.php
```

2) Replace section infos:

```text
NewProcedureSeeder
procedure name
scenario
steps
NPCs
actions
guidance
error keys
help keys
```

## Seeder Function Descriptions

Below are the functions, each of them creates one part of the training procedure data.

### `seedProcedure()`

Creates the main procedure record in the `procedures` table.

This is the high-level identity of the procedure: name, acronym, description, sector, group, supported modes, status, and config layer.


### `seedSteps()`

Creates the ordered checklist/flow for the procedure in the `steps` table.

Each step usually includes:

- `key`: machine-readable step name
- `name`: display name
- `description`: what the trainee must do
- `expected_action`: action keyword(s) expected from the student/AI
- `guidance_dialogue`: what the nurse/helper might say
- `error_keys`: possible mistakes
- `help_keys`: hints
- `is_terminal`: marks the final step

This is basically the procedure script.

### `seedScenario()`

Creates the environment/context for the procedure in the `scenarios` table.

A procedure can have one or more scenarios. For example, the same procedure could happen in a hospital room, emergency room, ICU, etc.

It also stores scenario settings such as patient status, environment flags, and behavior mode.

### `seedNpcs()`

Creates the characters/NPCs for the scenario.

Usually this includes:

- patient
- nurse/helper

It also creates NPC configurations, such as their AI prompt, personality, available context, model settings, and interaction behavior.

This tells the AI how each character should behave.

### `seedActions()`

Creates the allowed action commands for each NPC.

These are the fixed action keywords the AI can choose from, mapped to Unity animation commands.

Example:

```php
['hand_over_syringe', 'AnimHandOverSyringe', null, 'Hand the syringe to the trainee.']
```

Meaning:

- AI/internal keyword: `hand_over_syringe`
- Unity command: `AnimHandOverSyringe`
- optional args: `null`
- description: what the action does

This keeps the AI from inventing actions Unity cannot run.

### `seedSession()`

Creates a starting/open training session for testing.

It links together:

- user
- procedure
- scenario
- current first step
- session status
- session type

This lets the AI Manager immediately test turns without manually creating a session first.
