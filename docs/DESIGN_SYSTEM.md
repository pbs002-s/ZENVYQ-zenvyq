# Design System

Status: **placeholder until the user uploads the design.** When the design arrives, replace token values here and in `app/globals.css`; components should not need changes.

## Direction (until design is provided)
- Minimal, lots of white space, strong typography, fine detailing (thin borders, subtle grain, precise alignment).
- One accent colour, used sparingly for CTAs and the mascot's eyes.
- Light and dark themes for the site; portal follows system theme.

## Tokens (CSS variables in `app/globals.css`)
| Token | Placeholder value | Use |
|---|---|---|
| `--bg` | `#FAFAF9` | Page background |
| `--surface` | `#FFFFFF` | Cards |
| `--fg` | `#0C0C0D` | Body text |
| `--muted` | `#6B6B70` | Secondary text |
| `--border` | `#E7E5E4` | Hairlines |
| `--accent` | `#5B5BF6` | CTA, links, mascot eyes |
| `--accent-fg` | `#FFFFFF` | Text on accent |
| `--success` / `--warning` / `--danger` | `#16A34A` / `#D97706` / `#DC2626` | Status badges |
| `--radius` | `14px` | Cards, buttons |
| `--font-sans` | Inter Tight (Google Fonts via `next/font`) | UI + body |
| `--font-display` | Same family, tighter tracking | Headlines |
| Spacing | Tailwind 4 px scale | |
| Container | max 1200 px, 16 px gutter on mobile, 32 px desktop | |

Dark theme overrides the same tokens under `[data-theme="dark"]` and `prefers-color-scheme`.

## Type scale
| Name | Size (mobile → desktop) | Weight |
|---|---|---|
| Display | 44 → 88 px | 600 |
| H1 | 36 → 56 px | 600 |
| H2 | 28 → 40 px | 600 |
| H3 | 20 → 24 px | 600 |
| Body | 16 → 18 px | 400 |
| Small | 14 px | 400 |

## Site components
TopNav (open/transparent → solid on scroll), MobileMenu, SectionHeader, ServiceCard, WorkCard, LogoWall, StatItem, ProcessStep, TestimonialCard, PricingCard, FaqItem, ContactForm, Footer, CTAButton (primary/ghost).

## Portal components
Sidebar, Topbar, NotificationBell, DataTable (sort/filter/paginate), StatusBadge, RoleBadge, FileUpload (drag-drop, progress, type/size check), KanbanBoard/Column/Card, CommentThread, RichTextEditor, EmptyState, ConfirmDialog, Toast.

## Motion
- Section reveal: fade + 12 px rise, 400 ms, once.
- Nav background transition 200 ms.
- All motion disabled under `prefers-reduced-motion`.

## 3D mascot spec
**Working name:** "Zen" (rename in CMS / OPEN_QUESTIONS).
**Look:** original, small, chubby low-poly panda-like bot.
- Body: rounded box, off-white matte.
- Head: slightly larger sphere, dark visor band across the face with two glowing pill eyes in `--accent`.
- Ears: two small dark spheres (panda cue).
- Arms: short capsules for wave/point.
- Optional: tiny antenna with a glowing tip that pulses when it speaks.
- Not a copy of any existing brand mascot or logo.

**Poses (state machine):**
| State | Trigger | Animation |
|---|---|---|
| `idle` | default | gentle float (sin on y), slow blink, eyes track cursor |
| `wave` | page load / hero | arm wave ×2 + bubble "Hi! Want a quick tour?" |
| `point` | services / work in view | turns toward content, arm points |
| `think` | process / FAQ in view | head tilt, antenna pulse |
| `celebrate` | lead submitted | small jump + spin |
| `sleep` | 60 s no interaction | eyes dim to lines |

**Behaviour:** bottom-right, 140 px (100 px mobile), speech bubble above, click/Enter opens quick-nav. Bubble lines come from `site_settings.mascot.lines[sectionKey]`.

**Accessibility:** button with `aria-label="Open quick navigation"`, menu is a real list of links, Esc closes, focus returns to the mascot button. Canvas is `aria-hidden`.
