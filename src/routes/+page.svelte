<script lang="ts">
    import { browser } from "$app/environment";
    import { AppConfig, type Track } from "$lib/config";
    import { point } from "drizzle-orm/pg-core";
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

    let pointsToAdd: number | null = $state(null);

    interface ChosenTrack extends Track {
        index: number;
    }

    interface LocalStorageData {
        chosen_tracks: ChosenTrack[];
        started_at: number;
        score_delta: number;
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
            score_delta: 0,
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

        window.addEventListener("keypress", (e) => {
            if (e.key === "Enter") {
                if (mogi && pointsToAdd && pointsToAdd !== 0) {
                    mogi.score_delta =
                        (mogi?.score_delta || 0) + (pointsToAdd as number);
                    saveMogi(mogi);
                    pointsToAdd = null;
                }
            }
        });

        return () => {
            clearMogi();
        };
    });

    let showBanner = $state(true);
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
    <!-- {#if showBanner} -->
        <div class="banner" style:width={showBanner ? "calc(70% - 1em - 25px)" : "0px"} style:opacity={showBanner ? 1 : 0}>
            <Text weight="bold" maxLines={1} inheritColor>This site will not be updated with new tracks as I do not actively play Lounge.</Text>
            <Text weight="black" inheritColor><span onclick={() => {showBanner = false;}}>X</span></Text>
        </div>
    <!-- {/if} -->
    <div class="left">
        <Column justifyContent="center" alignItems="center" textAlign="left">
            <Column
                justifyContent="center"
                alignItems="center"
                textAlign="left"
            >
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
            <Column
                justifyContent="center"
                alignItems="center"
                textAlign="left"
            >
                <Heading size={5} weight="bold"
                    >Chosen Track{mogi?.chosen_tracks.length === 1 ? "" : "s"} ({mogi
                        ?.chosen_tracks.length || 0}{(mogi?.chosen_tracks
                        .length || 0) === 12
                        ? " - MOGI COMPLETE"
                        : ""})</Heading
                >

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
    </div>
    <div class="right">
        <Column
            justifyContent="flex-start"
            alignItems="flex-start"
            textAlign="left"
        >
            <Heading size={4} weight="bold">Score Tracker</Heading>
            <Text weight="semibold" sizeEm={0.9}
                >Track your disconnect/11p score delta.</Text
            >
            <Text weight="bold" sizeEm={1.3}
                >Current Delta: {(mogi?.score_delta || 0) >= 0
                    ? `+${mogi?.score_delta}`
                    : `${mogi?.score_delta}`}</Text
            >
            <HorizontalRule widthPercent={100} />
            <Row heightPx="fit" justifyContent="flex-start" gapEm={0.33}>
                <Button
                    size="large"
                    label="Delta +1"
                    type="primary"
                    onclick={() => {
                        if (mogi) {
                            mogi.score_delta = (mogi?.score_delta || 0) + 1;
                            saveMogi(mogi);
                            pointsToAdd = null;
                        }
                    }}
                />

                <Button
                    size="large"
                    label="Delta +2"
                    type="primary"
                    onclick={() => {
                        if (mogi) {
                            mogi.score_delta = (mogi?.score_delta || 0) + 2;
                            saveMogi(mogi);
                            pointsToAdd = null;
                        }
                    }}
                />

                <Button
                    size="large"
                    label="Delta +3"
                    type="primary"
                    onclick={() => {
                        if (mogi) {
                            mogi.score_delta = (mogi?.score_delta || 0) + 3;
                            saveMogi(mogi);
                            pointsToAdd = null;
                        }
                    }}
                />
            </Row>
            <Row heightPx="fit" justifyContent="flex-start" gapEm={0.33}>
                <Button
                    size="large"
                    label="Delta +4"
                    type="primary"
                    onclick={() => {
                        if (mogi) {
                            mogi.score_delta = (mogi?.score_delta || 0) + 4;
                            saveMogi(mogi);
                            pointsToAdd = null;
                        }
                    }}
                />

                <Button
                    size="large"
                    label="Delta +5"
                    type="primary"
                    onclick={() => {
                        if (mogi) {
                            mogi.score_delta = (mogi?.score_delta || 0) + 5;
                            saveMogi(mogi);
                            pointsToAdd = null;
                        }
                    }}
                />

                <Button
                    size="large"
                    label="Delta +6"
                    type="primary"
                    onclick={() => {
                        if (mogi) {
                            mogi.score_delta = (mogi?.score_delta || 0) + 6;
                            saveMogi(mogi);
                            pointsToAdd = null;
                        }
                    }}
                />
            </Row>
            <Row heightPx="fit" justifyContent="flex-start" gapEm={0.33}>
                <Button
                    size="large"
                    label="Delta -1"
                    type="secondary"
                    onclick={() => {
                        if (mogi) {
                            mogi.score_delta = (mogi?.score_delta || 0) - 1;
                            saveMogi(mogi);
                            pointsToAdd = null;
                        }
                    }}
                />

                <Button
                    size="large"
                    label="Delta -2"
                    type="secondary"
                    onclick={() => {
                        if (mogi) {
                            mogi.score_delta = (mogi?.score_delta || 0) - 2;
                            saveMogi(mogi);
                            pointsToAdd = null;
                        }
                    }}
                />

                <Button
                    size="large"
                    label="Delta -3"
                    type="secondary"
                    onclick={() => {
                        if (mogi) {
                            mogi.score_delta = (mogi?.score_delta || 0) - 3;
                            saveMogi(mogi);
                            pointsToAdd = null;
                        }
                    }}
                />
            </Row>
            <Row heightPx="fit" justifyContent="flex-start" gapEm={0.33}>
                <Button
                    size="large"
                    label="Copy Delta"
                    type="success"
                    onclick={() => {
                        if (mogi && browser) {
                            window.navigator.clipboard.writeText(
                                `${
                                    (mogi?.score_delta || 0) >= 0
                                        ? `+${mogi?.score_delta}`
                                        : `${mogi?.score_delta}`
                                }`,
                            );
                        }
                    }}
                />
                <Button
                    size="large"
                    label="Clear Delta"
                    type="danger"
                    onclick={() => {
                        if (mogi) {
                            mogi.score_delta = 0;
                            saveMogi(mogi);
                        }
                    }}
                />
            </Row>
            <HorizontalRule widthPercent={100} />
            <Text weight="bold">Add Custom Amount</Text>
            <Text weight="semibold" sizeEm={0.8} classList={["green"]}
                >Supports Negatives</Text
            >
            <input
                type="number"
                name="custom-amount"
                placeholder="Type a Number..."
                bind:value={
                    () => pointsToAdd,
                    (v) => {
                        if (v) {
                            pointsToAdd = Math.round(v);
                        } else pointsToAdd = v;
                    }
                }
            />
            {#if pointsToAdd !== null}
                <Button
                    label="Delta {pointsToAdd > 0 ? '+' : ''}{pointsToAdd}"
                    type="success"
                    onclick={() => {
                        if (mogi) {
                            mogi.score_delta =
                                (mogi?.score_delta || 0) +
                                (pointsToAdd as number);
                            saveMogi(mogi);
                            pointsToAdd = null;
                        }
                    }}
                />
            {/if}
            <HorizontalRule />
            <Heading size={5} weight="bold">Race Options</Heading>
            <Button
                label="Copy Chosen Track Emojis"
                onclick={copyEmojiString}
                size="normal"
                type="primary"
            />
        </Column>
    </div>
{/if}

<style lang="scss">
    .banner {
        transition: 0.3s all;
        position: absolute;
        left: 0;
        max-width: 70%;
        background-image: linear-gradient(to left, var(--yellow), var(--accent));
        color: var(--mantle) !important;
        display: flex;
        flex-direction: row;
        align-items: center;
        justify-content: space-between;
        padding: 0.33em 0.5em;
        border-radius: var(--border-md) 0 0px var(--border-md);
        & span {
            cursor: pointer;
        }
    }

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

    .right {
        position: absolute;
        /* transform: translate(-50%, -50%); */
        right: 0;
        bottom: 0;
        width: 30%;
        height: 90%;
        background-color: var(--mantle);
        padding: 2em 0.66em;
        border-right: 5px solid var(--crust);
    }

    .left {
        width: 70%;
    }

    input[type="number"] {
        appearance: textfield;
        -webkit-appearance: textfield;
        -moz-appearance: textfield;
        background-color: var(--surface-0);
        outline: none;
        border: 3px solid var(--surface-1);
        border-radius: var(--border-md);
        padding: 0.5em 1em;
        color: var(--text);
        font-size: 1em;
    }
</style>
