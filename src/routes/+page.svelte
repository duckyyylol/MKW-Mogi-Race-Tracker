<script lang="ts">
    import { browser } from "$app/environment";
    import { AppConfig, type Track } from "$lib/config";
    import {
        Button,
        Column,
        Heading,
        HorizontalRule,
        Row,
        Text,
    } from "duckylib";
    import { onMount } from "svelte";

    let mogiStarted = $state(false);
    let mogi: LocalStorageData | null = $state(null);
    let availableTracks: Track[] = $state([]);

    interface ChosenTrack extends Track {
        index: number;
    }

    interface LocalStorageData {
        chosen_tracks: ChosenTrack[];
        started_at: number;
    }

    function getMogi(): LocalStorageData | null {
        if (!browser) return null;
        try {
            let mogiString = window.localStorage.getItem("mogi") || null;
            if (!mogiString) return null;
            return JSON.parse(mogiString) as LocalStorageData;
        } catch (e) {
            return null;
        }
    }

    function saveMogi(mogiData: LocalStorageData): boolean {
        try {
            if (!browser) return false;

            let mogiString = JSON.stringify(mogiData);

            mogi = mogiData;
            window.localStorage.setItem("mogi", mogiString);
            return true;
        } catch (e) {
            return false;
        }
    }

    function addChosenTrack(trackId: string): boolean {
        try {
            if (!mogi) return false;
            if (mogi.chosen_tracks.length === 12) return false;
            if (!mogi?.chosen_tracks.some((t) => t.id === trackId)) {
                let foundTrack = AppConfig.tracks.find((t) => t.id === trackId);
                if (!foundTrack) return false;
                mogi.chosen_tracks = [
                    ...((mogi?.chosen_tracks || []) as any),
                    foundTrack,
                ];
            } else return false;

            saveMogi(mogi);
            return true;
        } catch (e) {
            return false;
        }
    }

    function removeChosenTrack(trackId: string): boolean {
        try {
            if (!mogi) return false;
            if (mogi.chosen_tracks.some((t) => t.id === trackId)) {
                let foundTrack = AppConfig.tracks.find((t) => t.id === trackId);
                if (!foundTrack) return false;
                mogi.chosen_tracks = mogi.chosen_tracks.filter(
                    (t) => t.id !== foundTrack.id,
                );
            } else return false;

            saveMogi(mogi);
            return true;
        } catch (e) {
            return false;
        }
    }

    function clearMogi() {
        if (browser) window.localStorage.removeItem("mogi");
        mogi = null;
        mogiStarted = false;
    }

    function initMogi(): LocalStorageData {
        if (!mogiStarted) mogiStarted = true;
        let data: LocalStorageData = {
            chosen_tracks: [],
            started_at: Date.now(),
        };

        mogi = data;
        saveMogi(mogi);

        return data;
    }

    function copyEmojiString(): void {
        if (browser) {
            let emojiStr = (mogi?.chosen_tracks || [])
                .map(
                    (t, i) =>
                        `<:_:${t.emoji_id}>${!((i + 1) % 6) ? "\n" : " "}`,
                )
                .join("");
            try {
                window.navigator.clipboard.writeText(emojiStr);
                // alert("Copied!");
            } catch (e) {}
        }
    }

    onMount(() => {
        clearMogi();
        initMogi();
        setInterval(() => {
            let mogiData = getMogi();
            if (mogiData) {
                mogi = mogiData;
                if (mogi && !mogiStarted) mogiStarted = true;
                availableTracks = AppConfig.tracks.filter(
                    (t) => !mogi?.chosen_tracks.some((c) => c.id === t.id),
                );
            }
        }, 1e3);

        return () => {
            clearMogi();
        };
    });
</script>

{#if !mogi && !mogiStarted}
    <Column justifyContent="center" alignItems="center" textAlign="left">
        <Heading size={4} weight="bold">Click to Start Mogi</Heading>
        <Button
            label="Start Mogi"
            onclick={initMogi}
            size="large"
            type="success"
        />
    </Column>
{:else}
    <Column justifyContent="center" alignItems="center" textAlign="left">
        <Column justifyContent="center" alignItems="center" textAlign="left">
            <Heading size={4} weight="bold">Available Tracks</Heading>

            <Row
                justifyContent="center"
                alignItems="center"
                textAlign="left"
                widthPercent={80}
                flexWrap={true}
            >
                {#each availableTracks as track}
                    <div
                        class="track"
                        style={mogi?.chosen_tracks.some(
                            (t) => t.id === track.id,
                        )
                            ? "display: none;"
                            : ""}
                        onclick={() => addChosenTrack(track.id)}
                    >
                        <img
                            title={track.name}
                            src={track.poster_path}
                            alt="{track.name} poster"
                            class="available"
                        />
                        <div class="popup">
                            <Column>
                                <Text weight="bold" sizeEm={0.9}
                                    >{track.name}</Text
                                >
                                <Text
                                    weight="bold"
                                    sizeEm={0.7}
                                    classList={["green"]}
                                    >Click to Mark Chosen</Text
                                >
                            </Column>
                        </div>
                    </div>
                {/each}
            </Row>
        </Column>
        <HorizontalRule widthPercent={90} />
        <Column justifyContent="center" alignItems="center" textAlign="left">
            <Heading size={5} weight="bold"
                >Chosen Track{mogi?.chosen_tracks.length === 1 ? "" : "s"} ({mogi
                    ?.chosen_tracks.length || 0}{(mogi?.chosen_tracks.length ||
                    0) === 12
                    ? " - MOGI COMPLETE"
                    : ""})</Heading
            >
            <Button
                label="Copy Track Emojis"
                onclick={copyEmojiString}
                size="large"
                type="primary"
            />
            <Row
                justifyContent="center"
                alignItems="center"
                textAlign="left"
                widthPercent={80}
                flexWrap={true}
            >
                {#each mogi?.chosen_tracks as track}
                    <div
                        class="track"
                        onclick={() => removeChosenTrack(track.id)}
                    >
                        <img
                            title={track.name}
                            src={track.poster_path}
                            alt="{track.name} poster"
                            class="grayscale chosen"
                        />
                        <div class="popup">
                            <Column>
                                <Text weight="bold" sizeEm={0.9}
                                    >{track.name}</Text
                                >
                            </Column>
                        </div>
                    </div>
                {/each}
            </Row>
        </Column>
    </Column>
{/if}

<style lang="scss">
    .grayscale {
        filter: grayscale();
    }
    img {
        cursor: pointer;
    }

    img.available,
    img.chosen {
        width: 7em;
        border-radius: 16px;
        position: relative;
        top: 0;
        left: 0;
    }

    img.available {
        border: 3px solid var(--green);
    }

    img.chosen {
        border: 3px solid var(--red);
    }

    .track {
        display: block;
        position: relative;
    }

    .popup {
        position: absolute;
        top: 5%;
        left: 50%;
        transform: translate(-50%, -50%);
        background-color: var(--mantle);
        display: none;
        padding: 0.66em 1.66em;
        border-radius: var(--border-md);
        border: 3px solid var(--accent);
    }

    /* .track {
        img.available,
        img.chosen {
            &:hover + .popup {
                display: block;
                z-index: 500;
            }

            &:not(:hover) + .popup {
                display: none;
                z-index: -100;
            }
        }
    } */

    img.available:hover,
    img.chosen:hover {
        transition: all 0.1s;
        scale: 2;
        z-index: 100;
    }

    img.available:not(:hover),
    img.chosen:not(:hover) {
        transition: all 0.3s;
        scale: 1;
    }
</style>
