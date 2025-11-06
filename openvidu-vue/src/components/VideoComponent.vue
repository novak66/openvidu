<script lang="ts">
import { LocalVideoTrack, RemoteVideoTrack } from 'livekit-client';
import { defineComponent } from 'vue';

type TrackProps = {
    track: LocalVideoTrack | RemoteVideoTrack;
    participantIdentity: string;
    local?: boolean;
};

export default defineComponent({
    props: {
        track: {
            type: Object as () => LocalVideoTrack | RemoteVideoTrack,
            required: true
        },
        participantIdentity: {
            type: String,
            required: true
        },
        local: {
            type: Boolean,
            default: false
        },
        audio: {
            type: Boolean,
            default: true
        },
        video: {
            type: Boolean,
            default: true
        }
    },

    emits: ['change-camera', 'change-microphone'],

    mounted() {
        this.attachTrack();
    },

    watch: {
        track(newTrack, oldTrack) {
            if (oldTrack) {
                oldTrack.detach();
            }

            if (newTrack) {
                this.attachTrack();
            }
        }
    },

    methods: {
        attachTrack() {
            const videoElement = this.$refs.videoElement as HTMLMediaElement;

            if (!this.track) {
                return;
            }

            if (!videoElement) {
                return;
            }

            try {
                this.track.attach(videoElement);
                console.log('✅ Vídeo anexado:', this.participantIdentity);
            } catch (error) {
                console.error('❌ Erro ao anexar vídeo:', error);
            }
        },

        handleCameraClick() {
            console.log('🔵 Clique detectado no VideoComponent');
            this.$emit('change-camera');
        }
    },

    beforeDestroy() {
        this.track.detach();
    }
});
</script>

<template>
    <div :id="'camera-' + participantIdentity" class="video-container">
        <div class="participant-data">
            <p>{{ participantIdentity + (local ? ' (You)' : '') }}</p>
        </div>
        <div
            style="
                position: absolute;
                bottom: 0;
                display: flex;
                justify-content: space-around;
                width: 100%;
                z-index: 9999;
            "
        >
            <div style="cursor: pointer" @click="handleCameraClick()">
                <div
                    v-if="video"
                    style="
                        width: 40px;
                        height: 40px;
                        cursor: pointer;
                        background-color: white;
                        border-radius: 50%;
                        display: flex;
                        align-items: center;
                        justify-content: center;
                        margin-bottom: 10px;
                    "
                >
                    <svg viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
                        <path
                            class="active"
                            d="M17 10.5V7c0-.55-.45-1-1-1H4c-.55 0-1 .45-1 1v10c0 .55.45 1 1 1h12c.55 0 1-.45 1-1v-3.5l4 4v-11l-4 4zM15 16H5V8h10v8z"
                        />
                    </svg>
                </div>
                <div
                    v-else
                    style="
                        width: 40px;
                        height: 40px;
                        cursor: pointer;
                        background-color: white;
                        border-radius: 50%;
                        display: flex;
                        align-items: center;
                        justify-content: center;
                        margin-bottom: 10px;
                    "
                >
                    <svg viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
                        <path
                            class="inactive"
                            d="M21 6.5l-4 4V7c0-.55-.45-1-1-1H9.82L21 17.18V6.5zM3.27 2L2 3.27 4.73 6H4c-.55 0-1 .45-1 1v10c0 .55.45 1 1 1h12c.21 0 .39-.08.54-.18L19.73 21 21 19.73 3.27 2zM15 16H5V8h.73L15 17.27V16z"
                        />
                    </svg>
                </div>
            </div>
            <div style="cursor: pointer" @click="$emit('change-microphone')">
                <div
                    v-if="audio"
                    style="
                        width: 40px;
                        height: 40px;
                        cursor: pointer;
                        background-color: white;
                        border-radius: 50%;
                        display: flex;
                        align-items: center;
                        justify-content: center;
                        margin-bottom: 10px;
                    "
                >
                    <svg viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
                        <path
                            class="active"
                            d="M12 14c1.66 0 3-1.34 3-3V5c0-1.66-1.34-3-3-3S9 3.34 9 5v6c0 1.66 1.34 3 3 3z"
                        />
                        <path
                            class="active"
                            d="M17 11c0 2.76-2.24 5-5 5s-5-2.24-5-5H5c0 3.53 2.61 6.43 6 6.92V21h2v-3.08c3.39-.49 6-3.39 6-6.92h-2z"
                        />
                    </svg>
                </div>
                <div
                    v-else
                    style="
                        width: 40px;
                        height: 40px;
                        cursor: pointer;
                        background-color: white;
                        border-radius: 50%;
                        display: flex;
                        align-items: center;
                        justify-content: center;
                        margin-bottom: 10px;
                    "
                >
                    <svg viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
                        <path
                            class="inactive"
                            d="M19 11h-1.7c0 .74-.16 1.43-.43 2.05l1.23 1.23c.56-.98.9-2.09.9-3.28zm-4.02.17c0-.06.02-.11.02-.17V5c0-1.66-1.34-3-3-3S9 3.34 9 5v.18l5.98 5.99zM4.27 3L3 4.27l6.01 6.01V11c0 1.66 1.33 3 2.99 3 .22 0 .44-.03.65-.08l1.66 1.66c-.71.33-1.5.52-2.31.52-2.76 0-5.3-2.1-5.3-5.1H5c0 3.41 2.72 6.23 6 6.72V21h2v-3.28c.91-.13 1.77-.45 2.54-.9L19.73 21 21 19.73 4.27 3z"
                        />
                    </svg>
                </div>
            </div>
        </div>
        <video ref="videoElement"></video>
    </div>
</template>

<style scoped>
.video-container {
    position: relative;
    background: #3b3b3b;
    aspect-ratio: 16/9;
    border-radius: 6px;
    overflow: hidden;
}

.video-container video {
    width: 100%;
    height: 100%;
}

.video-container .participant-data {
    position: absolute;
    top: 0;
    left: 0;
}

.participant-data p {
    background: #f8f8f8;
    margin: 0;
    padding: 0 5px;
    color: #777777;
    font-weight: bold;
    border-bottom-right-radius: 4px;
}

/* Media Queries */
@media screen and (max-width: 480px) {
    .video-container {
        aspect-ratio: 9/16;
    }
}

body {
    font-family: Arial, sans-serif;
    padding: 40px;
    background: #f5f5f5;
    display: flex;
    flex-direction: column;
    align-items: center;
}
.container {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 30px;
    margin-top: 20px;
}
.icon-box {
    background: white;
    padding: 30px;
    border-radius: 10px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
    text-align: center;
}
.icon-box h3 {
    margin: 15px 0 0 0;
    color: #333;
    font-size: 14px;
}
svg {
    width: 30px;
    height: 30px;
}
.active {
    fill: #4caf50;
}
.inactive {
    fill: #f44336;
}
</style>
