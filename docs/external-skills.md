# Skills de terceros (dependencias externas)

## Qué es y por qué

El hub no versiona skills de terceros:

- 11 de ellas no declaran licencia.
- `gsap-plugins` tiene 433 líneas frente al límite de 300.
- Cambian con upstream y se desincronizan.

En su lugar, el hub registra origen, licencia y comando de reinstalación. El manifiesto es `config/external-skills.json`. Máquina de referencia: `~/.claude/skills`; fecha de observación: 2026-11-08.

## HyperFrames

- Repo: https://github.com/heygen-com/hyperframes
- Licencia: Apache-2.0, según su README.
- Upstream publica 9 skills.

| Skill | Publicada upstream (sí/no) |
| --- | --- |
| hyperframes | sí |
| hyperframes-cli | sí |
| remotion-to-hyperframes | sí |
| animejs | no |
| css-animations | no |
| gsap | no |
| lottie | no |
| three | no |
| waapi | no |
| tailwind | no |
| website-to-hyperframes | no |

Las 8 no publicadas no figuran entre las skills publicadas hoy por upstream (posiblemente obsoletas o retiradas), y su origen no está confirmado.

## GSAP oficial

- Repo: https://github.com/greensock/gsap-skills
- Licencia: MIT, según el frontmatter local de los SKILL.md.
- Esta fuente NO está verificada contra el README upstream: el repo y el comando proceden de resultados de búsqueda.
- Skills: gsap-core, gsap-frameworks, gsap-performance, gsap-plugins, gsap-react, gsap-scrolltrigger, gsap-timeline, gsap-utils.

## Reinstalar

```bash
pnpm dlx skills add heygen-com/hyperframes --all
pnpm dlx hyperframes skills update
pnpm dlx skills add https://github.com/greensock/gsap-skills
```

Upstream documenta estos comandos con `npx`; el hub usa `pnpm dlx`. Los comandos no se han ejecutado aquí.

## Cómo verificar

Compara `ls ~/.claude/skills` con los nombres del manifiesto:

```bash
jq -r '.sources[].skills[].name' config/external-skills.json | sort
```
