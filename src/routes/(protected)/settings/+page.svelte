<script lang="ts">
    import type { ActionData, PageData } from "./$types";
    import { BasicForm } from "@sjsf/form";
    import { createMeta, setupSvelteKitForm } from "@sjsf/sveltekit/client";
    import * as defaults from "$lib/components/settings/form-defaults";
    import { setShadcnContext } from "$lib/components/shadcn-context";
    import { toast } from "svelte-sonner";
    import { icons } from "@sjsf/lucide-icons";
    setShadcnContext();

    let { data }: { data: PageData } = $props();

    const categoryRules = [
        {
            id: "general",
            title: "General",
            description: "Core options that keep Riven running smoothly and securely.",
            keywords: ["general", "app", "server", "privacy", "base", "auth"]
        },
        {
            id: "scrapers",
            title: "Scrapers & Sources",
            description: "Providers, scrapers, and indexers that feed results into Riven.",
            keywords: ["scraper", "provider", "indexer", "torrent", "torznab", "prowlarr", "jackett", "debrid", "torrentio", "source", "orion", "comet"]
        },
        {
            id: "filters",
            title: "Filters & Ranking",
            description: "Preferences that shape matching, filtering, and quality scoring.",
            keywords: ["filter", "rank", "quality", "resolution", "language", "rtn", "match", "preferred", "exclude", "require", "custom"]
        },
        {
            id: "integrations",
            title: "Integrations & Notifications",
            description: "Connections to media servers, webhooks, and notification services.",
            keywords: ["integration", "notification", "webhook", "radarr", "sonarr", "overseerr", "plex", "emby", "jellyfin", "discord", "telegram"]
        },
        {
            id: "advanced",
            title: "Advanced",
            description: "Tuning for performance, caching, logging, and rate limits.",
            keywords: ["advanced", "cache", "log", "timeout", "ratelimit", "network", "retries", "limiter"]
        }
    ];

    let categorized = [] as {
        id: string;
        title: string;
        description: string;
        fields: string[];
    }[];

    const normalize = (value?: string) => value?.toLowerCase() ?? "";

    const categorizeSchema = () => {
        const properties = Object.entries((data.form?.schema as any)?.properties ?? {});
        const assigned = new Set<string>();

        const mapped = categoryRules.map((rule) => {
            const fields = properties
                .filter(([key, schema]) => {
                    const haystack = `${normalize(key)} ${normalize((schema as any)?.title)}`;
                    return rule.keywords.some((keyword) => haystack.includes(keyword));
                })
                .map(([key]) => {
                    assigned.add(key);
                    return key;
                });

            return { ...rule, fields };
        });

        const unassigned = properties
            .filter(([key]) => !assigned.has(key))
            .map(([key]) => key);

        categorized = [
            ...mapped.filter((rule) => rule.fields.length > 0),
            ...(unassigned.length
                ? [
                      {
                          id: "other",
                          title: "Other",
                          description: "Additional options that do not yet belong to a category.",
                          fields: unassigned
                      }
                  ]
                : [])
        ];
    };

    $: categorizeSchema();

    const meta = createMeta<ActionData, PageData>().form;

    // @ts-expect-error - Schema is provided by page data
    const { form, request } = setupSvelteKitForm(meta, {
        ...defaults,
        icons,
        delayedMs: 500,
        timeoutMs: 30000,
        onSuccess: (result) => {
            if (result.type === "success") {
                toast.success("Settings saved");
            } else {
                toast.error("Failed to save settings");
            }
        },
        onFailure: () => {
            toast.error("Something went wrong while saving settings");
        }
    });
</script>

<svelte:head>
    <title>Settings - Riven</title>
</svelte:head>

<div class="mt-14 h-full w-full px-6 pb-12 pt-6 md:px-12">
    <div class="mx-auto grid max-w-6xl gap-6 lg:grid-cols-[320px,1fr]">
        <section class="space-y-4 rounded-2xl border bg-card/60 p-6 shadow-sm backdrop-blur">
            <div class="space-y-3">
                <div class="space-y-1">
                    <p class="text-sm font-semibold text-foreground">Quick tips</p>
                    <p class="text-sm text-muted-foreground">
                        Settings are validated before saving. Check the field descriptions and category summaries to see
                        where to adjust scrapers, filters, integrations, and general behavior.
                    </p>
                </div>

                <div class="rounded-xl border bg-background/50 p-4">
                    <ul class="grid gap-3 text-sm text-muted-foreground">
                        <li class="flex gap-2">
                            <span class="mt-1.5 h-1.5 w-1.5 rounded-full bg-primary"></span>
                            <span>Use the category cards to jump straight to scrapers, filters, or general controls.</span>
                        </li>
                        <li class="flex gap-2">
                            <span class="mt-1.5 h-1.5 w-1.5 rounded-full bg-primary"></span>
                            <span>Save once after reviewing each category. Validation keeps edits safe before submission.</span>
                        </li>
                        <li class="flex gap-2">
                            <span class="mt-1.5 h-1.5 w-1.5 rounded-full bg-primary"></span>
                            <span>Experiment freely; errors will highlight only the fields that need attention.</span>
                        </li>
                    </ul>
                </div>

                <div class="space-y-2 rounded-xl border bg-background/60 p-4">
                    <div class="flex items-center justify-between gap-3">
                        <div class="space-y-1">
                            <p class="text-sm font-semibold">Categories</p>
                            <p class="text-xs text-muted-foreground">Grouped by common workflows.</p>
                        </div>
                        <div class="rounded-full bg-primary/10 px-3 py-1 text-xs font-medium text-primary">Guided</div>
                    </div>

                    <div class="grid gap-3">
                        {#if categorized.length === 0}
                            <p class="text-sm text-muted-foreground">Loading categories from the schema...</p>
                        {:else}
                            {#each categorized as category (category.id)}
                                <button
                                    type="button"
                                    class="flex w-full flex-col gap-1 rounded-xl border bg-background/80 px-3 py-3 text-left shadow-sm transition hover:border-primary/70 hover:bg-background"
                                    on:click={() => {
                                        const target = category.fields
                                            .map((field) => {
                                                const id = `root__${field}`;
                                                return (
                                                    document.getElementById(id) ??
                                                    document.querySelector(`[id*="${field}"]`) ??
                                                    null
                                                );
                                            })
                                            .find((el) => Boolean(el)) as HTMLElement | null;

                                        target?.scrollIntoView({ behavior: "smooth", block: "start" });
                                    }}
                                >
                                    <div class="flex items-center justify-between gap-2">
                                        <p class="text-sm font-semibold text-foreground">{category.title}</p>
                                        <span class="rounded-full bg-primary/10 px-2 py-0.5 text-[11px] font-medium text-primary">
                                            {category.fields.length} field{category.fields.length === 1 ? "" : "s"}
                                        </span>
                                    </div>
                                    <p class="text-xs text-muted-foreground">{category.description}</p>

                                    {#if category.fields.length}
                                        <div class="flex flex-wrap gap-2">
                                            {#each category.fields as field}
                                                <span class="rounded-full bg-muted px-2 py-0.5 text-[11px] font-medium text-foreground/80">
                                                    {field.replaceAll("_", " ")}
                                                </span>
                                            {/each}
                                        </div>
                                    {/if}
                                </button>
                            {/each}
                        {/if}
                    </div>
                </div>
            </div>
        </section>

        <section class="space-y-4 rounded-2xl border bg-card/60 p-6 shadow-sm backdrop-blur">
            <header class="space-y-2">
                <p class="text-sm font-medium text-primary">Workspace</p>
                <h1 class="text-3xl font-semibold tracking-tight">Settings</h1>
                <p class="text-sm text-muted-foreground">
                    Update how Riven handles lookups, providers, and privacy without leaving the page. The form below
                    stays in sync with your current configuration.
                </p>
            </header>

            <div class="rounded-xl border bg-background/60 p-4">
                <p class="text-sm text-muted-foreground">
                    Changes are kept in the form until you save, so you can experiment freely. If anything is missing or
                    invalid, the field will point it out so you can fix it quickly. Use the category chips below to focus
                    on scrapers, filters, integrations, or general options.
                </p>
            </div>

            <div class="flex flex-wrap gap-2">
                {#each categorized as category (category.id)}
                    <span class="rounded-full border bg-background/70 px-3 py-1 text-xs font-medium text-foreground">
                        {category.title}
                    </span>
                {/each}
            </div>

            <div class="overflow-hidden rounded-2xl border bg-background/70 p-1 shadow-inner">
                <BasicForm {form} method="POST" class="grid gap-4 p-2 md:p-4"></BasicForm>
            </div>
        </section>
    </div>
</div>
