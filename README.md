# Zephyr RTOS AI Skills

Agent skills that help AI coding assistants work on [Zephyr RTOS](https://zephyrproject.org) projects the way Zephyr expects. They cover devicetree overlays and pin control on real vendor silicon, and Kconfig fragments that actually take effect.

## Why

General-purpose assistants often get Zephyr wrong in ways that cost hours:

- They mix up Linux and Zephyr devicetree conventions.
- They guess pin macros for the wrong SoC family.
- They set Kconfig options that are silently ignored.
- They miss changes between Zephyr releases.

These skills give the assistant Zephyr-specific knowledge and a working method:

- read your project and build output first
- apply the rules for your vendor and your Zephyr version
- verify the result with a build

## Skills

| Skill | Helps with |
| --- | --- |
| [`zephyr-devicetree`](skills/zephyr-devicetree/) | Overlays and board files, bindings, enabling peripherals and adding bus devices, pin control (pinctrl) with vendor-specific pages, `/chosen` and `/aliases`, flash partitions, reading devicetree from C, inspecting build output, sysbuild, and diagnosing devicetree errors |
| [`zephyr-kconfig`](skills/zephyr-kconfig/) | `prj.conf` and configuration fragments, merge order, finding the right option, dependencies (`depends on`, `select`, `imply`), "assigned y but got n" and other Kconfig errors, writing Kconfig for applications and modules, `CONFIG_` symbols in C, and sysbuild configuration |

The assistant loads each skill automatically when a task matches. You don't need to invoke it by name.

More skills are planned: application development, build and west workflows, subsystems, and user mode.

## Vendor coverage

The skills are generic by design. Where hardware differs, they add vendor pages:

- STMicroelectronics STM32
- NXP (MCX, Kinetis, LPC, i.MX RT, S32)
- Espressif ESP32
- Silicon Labs Series 2
- Nordic Semiconductor nRF

Other vendors are handled by a generic method that follows the board's own files. More vendor pages will follow.

## Zephyr versions

The skills are written against **Zephyr 4.4** and cover the differences for other releases, from **3.7 LTS** to **4.5**. The assistant checks which Zephyr version your project uses and follows the matching procedure.

## Principles

- **Grounded in your project.** Answers come from your overlays, configuration files, and build output, not from memory.
- **Verified.** Each rule is checked against the Zephyr source and documentation of the release it applies to.
- **Agent-friendly.** The skills only use commands an assistant can run on its own, such as `west build`, and read build artifacts. Interactive tools like menuconfig are left to you.
- **Tool-agnostic.** A Zephyr workspace, `west`, and a shell are enough. If they are available, the skills can also use editor tooling such as the Zephyr Workbench MCP server or a Zephyr documentation search tool.

## Installation

The skills use the open [Agent Skills](https://agentskills.io) format, so they work with most AI coding agents. Pick the method that fits your agent.

### Any agent, one command

```sh
npx skills add Ac6Embedded/Zephyr-RTOS-AI-Skills
```

This opens a picker where you choose the skills and the agents to install them for. It uses the open-source [skills CLI](https://github.com/vercel-labs/skills) and needs Node.js. Non-interactive examples:

```sh
# All skills, for all your projects, for the agents you use
npx skills add Ac6Embedded/Zephyr-RTOS-AI-Skills -g -a claude-code -a codex -a cursor -a github-copilot -y

# One skill only
npx skills add Ac6Embedded/Zephyr-RTOS-AI-Skills --skill zephyr-kconfig -g -a claude-code -y
```

Update later with `npx skills update`. With the GitHub CLI (2.90 or newer), `gh skill install Ac6Embedded/Zephyr-RTOS-AI-Skills --all --agent <agent>` does the same.

### Claude Code plugin

Add this repository as a plugin marketplace once, then install all the skills or only the ones you need:

```
/plugin marketplace add Ac6Embedded/Zephyr-RTOS-AI-Skills
/plugin install zephyr-skills@zephyr-rtos-ai-skills
/plugin install zephyr-devicetree@zephyr-rtos-ai-skills
/plugin install zephyr-kconfig@zephyr-rtos-ai-skills
```

Choose either the full bundle (`zephyr-skills`) or individual skills. Installing both loads the same skill twice. Get new skills with `/plugin marketplace update zephyr-rtos-ai-skills`.

### Manual copy

Each skill is a plain folder with a `SKILL.md`. Copy the folders into the skills directory of your agent:

```sh
git clone https://github.com/Ac6Embedded/Zephyr-RTOS-AI-Skills.git
mkdir -p ~/.agents/skills
cp -R Zephyr-RTOS-AI-Skills/skills/* ~/.agents/skills/
```

To install a single skill, copy only its folder, for example `skills/zephyr-kconfig`. Use the personal folder for all your projects, or the project folder to share the skills with a repository:

| Agent | Personal folder | Project folder |
| --- | --- | --- |
| Codex, GitHub Copilot (VS Code, CLI, cloud agent), Cursor, Gemini CLI, Windsurf, Devin, JetBrains Junie, OpenCode, Goose, Amp, Cline, Factory Droid, Mistral Vibe, OpenHands | `~/.agents/skills/` | `.agents/skills/` |
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| Kiro | `~/.kiro/skills/` | `.kiro/skills/` |

Keep one copy of each skill per location: some agents read several of these folders.

### Claude apps (claude.ai, desktop, mobile)

Zip each skill folder, then upload it in Claude under **Customize > Skills**. Code execution must be enabled in your settings.

```sh
cd Zephyr-RTOS-AI-Skills/skills
zip -r zephyr-devicetree.zip zephyr-devicetree
zip -r zephyr-kconfig.zip zephyr-kconfig
```

## Works well with

- [Zephyr Workbench](https://github.com/Ac6Embedded/vscode-zephyr-workbench), a VS Code extension for Zephyr development
- [Devicetree Manager for Zephyr](https://github.com/Ac6Embedded/vscode-devicetree-manager-for-zephyr), a visual devicetree and pin configuration editor

## Training

To learn AI-assisted embedded development or Zephyr itself, Ac6 offers these courses:

- [AI-Assisted Embedded Development](https://www.ac6-training.com/en/ai1/ai-assisted-embedded-development)
- [Zephyr RTOS Programming](https://www.ac6-training.com/en/rt5/zephyr-rtos-programming)

## Contributing

Contributions are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) for the authoring conventions and verification policy.

## License

Licensed under the [Apache License 2.0](LICENSE), the same license as Zephyr.

---

Maintained by [Ac6](https://www.ac6.fr). This is an independent project, not an official deliverable of the Zephyr Project. Zephyr is a trademark of the Linux Foundation.
