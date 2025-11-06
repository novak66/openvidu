<template>
    <audio v-if="track" ref="audioElement" autoplay></audio>
</template>

<script lang="ts">
import { LocalAudioTrack, RemoteAudioTrack } from 'livekit-client';

export default {
    props: {
        track: {
            type: Object as () => LocalAudioTrack | RemoteAudioTrack | undefined,
            required: false,
            default: undefined
        }
    },

    data() {
        return {
            isAttached: false
        };
    },

    mounted() {
        this.attachTrack();
    },

    updated() {
        this.attachTrack();
    },

    methods: {
        attachTrack() {
            if (!this.track || this.isAttached) {
                return;
            }

            const audioElement = this.$refs.audioElement as HTMLMediaElement;
            if (!audioElement) {
                return;
            }

            try {
                this.track.attach(audioElement);
                this.isAttached = true;
                console.log('✅ Áudio anexado');
            } catch (error) {
                console.error('❌ Erro ao anexar áudio:', error);
            }
        }
    },

    beforeDestroy() {
        if (this.track && this.isAttached) {
            this.track.detach();
            this.isAttached = false;
        }
    }
};
</script>
