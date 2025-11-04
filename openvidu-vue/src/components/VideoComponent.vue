<script lang="ts">
import { LocalVideoTrack, RemoteVideoTrack } from 'livekit-client';

type TrackProps = {
    track: LocalVideoTrack | RemoteVideoTrack;
    participantIdentity: string;
    local?: boolean;
};

export default {
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
        }
    },

    mounted() {
        const videoElement = this.$refs.videoElement as HTMLMediaElement;

        if (videoElement) {
            this.track.attach(videoElement);
        }
    },

    beforeDestroy() {
        this.track.detach();
    }
};
</script>

<template>
    <div :id="'camera-' + participantIdentity" class="video-container">
        <div class="participant-data">
            <p>{{ participantIdentity + (local ? ' (You)' : '') }}</p>
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
</style>
