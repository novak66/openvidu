<script lang="ts">
import {
    LocalVideoTrack,
    RemoteParticipant,
    RemoteTrack,
    RemoteTrackPublication,
    Room,
    RoomEvent
} from 'livekit-client';

import VideoComponent from './components/VideoComponent.vue';
import AudioComponent from './components/AudioComponent.vue';

type TrackInfo = {
    trackPublication: RemoteTrackPublication;
    participantIdentity: string;
};

export default {
    name: 'YourComponentName',

    components: {
        VideoComponent,
        AudioComponent
    },

    data() {
        return {
            APPLICATION_SERVER_URL: '',
            LIVEKIT_URL: '',
            APPLICATION_OPENVIDU_INTERNAL_SERVER_URL: '',
            APPLICATION_OPENVIDU_V1_SERVER_URL: '',
            participantName: 'Participant' + Math.floor(Math.random() * 100),
            roomName: 'Room',
            loginToken: '' as string,
            rooms: [] as Room[],
            room: undefined as Room | undefined,
            roomSelected: null as Room | null,
            localTrack: undefined as LocalVideoTrack | undefined,
            remoteTracksMap: new Map<string, TrackInfo>(),
            entrou: false as Boolean
        };
    },

    created() {
        this.configureUrls();
    },

    mounted() {
        this.login('admin', 'xlCvppgl2OxMOXsYP9SEtwhpAgPDZNLseVtp2Sd6X1X')
            .then((token) => {
                this.loginToken = token;
                return this.getRooms();
            })
            .then((rooms) => {
                this.rooms = rooms;
            })
            .catch((error) => {
                console.error('Erro ao iniciar:', error);
            });

        window.addEventListener('beforeunload', this.leaveRoom);
    },

    beforeDestroy() {
        this.leaveRoom();
        window.removeEventListener('beforeunload', this.leaveRoom);
    },

    methods: {
        configureUrls() {
            this.APPLICATION_SERVER_URL = 'https://' + 'hml-openvidu.prontumais.com.br/api/';
            this.APPLICATION_OPENVIDU_INTERNAL_SERVER_URL =
                'https://' + 'hml-openvidu.prontumais.com.br/internal-api/v1/';
            this.APPLICATION_OPENVIDU_V1_SERVER_URL = 'https://' + 'hml-openvidu.prontumais.com.br/api/v1/';
            this.LIVEKIT_URL = 'wss://' + 'hml-openvidu.prontumais.com.br';
        },

        async joinRoom() {
            this.room = new Room();

            this.room.on(
                RoomEvent.TrackSubscribed,
                (_track: RemoteTrack, publication: RemoteTrackPublication, participant: RemoteParticipant) => {
                    this.remoteTracksMap.set(publication.trackSid, {
                        trackPublication: publication,
                        participantIdentity: participant.identity
                    });
                }
            );

            this.room.on(RoomEvent.TrackUnsubscribed, (_track: RemoteTrack, publication: RemoteTrackPublication) => {
                this.remoteTracksMap.delete(publication.trackSid);
            });

            try {
                const token = await this.getToken(this.participantName);
                await this.room.connect(this.LIVEKIT_URL, token);
                await this.room.localParticipant.enableCameraAndMicrophone();

                const iterator = this.room.localParticipant.videoTrackPublications.values();
                const firstPublication = iterator.next().value;
                this.localTrack = firstPublication ? firstPublication.videoTrack : undefined;
                this.entrou = true;
            } catch (error: any) {
                console.log('Erro ao conectar na sala:', error.message);
                await this.leaveRoom();
            }
        },

        async leaveRoom() {
            if (this.room) {
                await this.room.disconnect();
            }

            this.room = undefined;
            this.localTrack = undefined;
            this.remoteTracksMap.clear();
        },

        async getToken(participantName: string): Promise<string> {
            const select = this.roomSelected;
            const response = await fetch(this.APPLICATION_OPENVIDU_INTERNAL_SERVER_URL + 'participants/token', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify({ roomId: select?.roomId, participantName, secret: '975734f820' })
            });

            if (!response.ok) {
                const error = await response.json();
                throw new Error(`Falha ao obter token: ${error.errorMessage}`);
            }

            const data = await response.json();
            return data.token;
        },

        async login(username: string, password: string): Promise<string> {
            const response = await fetch(this.APPLICATION_OPENVIDU_INTERNAL_SERVER_URL + 'auth/login', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ username, password })
            });

            if (!response.ok) {
                const error = await response.json();
                throw new Error(`Falha no login: ${error.errorMessage}`);
            }

            const data = await response.json();
            return data.accessToken;
        },

        async createRoom(roomName: string): Promise<any> {
            const response = await fetch(this.APPLICATION_SERVER_URL + 'rooms', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ roomName })
            });

            if (!response.ok) {
                const error = await response.json();
                throw new Error(`Falha ao criar sala: ${error.errorMessage}`);
            }

            return await response.json();
        },

        async getRooms(): Promise<Room[]> {
            const response = await fetch(this.APPLICATION_OPENVIDU_V1_SERVER_URL + 'rooms?maxItems=50', {
                method: 'GET',
                headers: {
                    'Content-Type': 'application/json',
                    Authorization: 'Bearer ' + this.loginToken
                }
            });

            if (!response.ok) {
                const error = await response.json();
                throw new Error(`Falha ao obter salas: ${error.errorMessage}`);
            }

            const data = await response.json();
            return data.rooms;
        }
    }
};
</script>

<template>
    <div>
        <div style="width: 100%" v-if="roomSelected == null">
            <div style="display: flex; align-items: center" v-for="r in rooms">
                <div>{{ r.roomName }}</div>
                <button class="btn btn-lg" @click="roomSelected = r">Entrar na sala</button>
            </div>
            <button class="btn btn-lg btn-success" @click="createRoom('room')">Criar sala</button>
        </div>
        <div v-else>
            <div v-if="!entrou" id="join">
                <div id="join-dialog">
                    <h2>Join a Video Room</h2>
                    <div>
                        <label for="participant-name">Participant</label>
                        <input
                            v-model="participantName"
                            id="participant-name"
                            class="form-control"
                            type="text"
                            required
                        />
                    </div>
                    <div>
                        <label for="room-name">Room</label>
                        <input
                            v-model="roomSelected.roomName"
                            id="room-name"
                            class="form-control"
                            type="text"
                            disabled
                        />
                    </div>
                    <button
                        class="btn btn-lg btn-success"
                        @click="joinRoom()"
                        :disabled="!roomName || !participantName"
                    >
                        Join!
                    </button>
                </div>
            </div>
            <div v-else id="room">
                <div id="room-header">
                    <h2 id="room-title">{{ roomName }}</h2>
                    <button class="btn btn-danger" id="leave-room-button" @click="leaveRoom">Leave Room</button>
                </div>
                <div id="layout-container">
                    <VideoComponent
                        v-if="localTrack"
                        :track="localTrack"
                        :participantIdentity="participantName"
                        :local="true"
                    />
                    <template
                        v-for="remoteTrack of remoteTracksMap.values()"
                        :key="remoteTrack.trackPublication.trackSid"
                    >
                        <VideoComponent
                            v-if="remoteTrack.trackPublication.kind === 'video'"
                            :track="remoteTrack.trackPublication.videoTrack!"
                            :participantIdentity="remoteTrack.participantIdentity"
                        />
                        <AudioComponent v-else :track="remoteTrack.trackPublication.audioTrack!" hidden />
                    </template>
                </div>
            </div>
        </div>
    </div>
</template>

<style scoped>
#join {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    height: 100%;
}

#join-dialog {
    width: 70%;
    max-width: 900px;
    padding: 60px;
    border-radius: 6px;
    background-color: #f0f0f0;
}

#join-dialog h2 {
    color: #4d4d4d;
    font-size: 60px;
    font-weight: bold;
    text-align: center;
}

#join-dialog form {
    text-align: left;
}

#join-dialog label {
    display: block;
    margin-bottom: 10px;
    color: #0088aa;
    font-weight: bold;
    font-size: 20px;
}

.form-control {
    width: 100%;
    padding: 8px;
    margin-bottom: 10px;
    box-sizing: border-box;
    color: #0088aa;
    font-weight: bold;
}

.form-control:focus {
    color: #0088aa;
    border-color: #0088aa;
    -webkit-box-shadow:
        inset 0 1px 1px rgba(0, 0, 0, 0.075),
        0 0 8px rgba(0, 136, 170, 0.6);
    box-shadow:
        inset 0 1px 1px rgba(0, 0, 0, 0.075),
        0 0 8px rgba(0, 136, 170, 0.6);
}

#join-dialog button {
    display: block;
    margin: 20px auto 0;
}

.btn {
    font-weight: bold;
}

.btn-success {
    background-color: #06d362;
    border-color: #06d362;
}

.btn-success:hover {
    background-color: #1abd61;
    border-color: #1abd61;
}

#room {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
}

#room-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    width: 100%;
    max-width: 1000px;
    padding: 0 20px;
    margin-bottom: 20px;
}

#room-title {
    font-size: 2em;
    font-weight: bold;
    margin: 0;
}

#layout-container {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
    gap: 10px;
    justify-content: center;
    align-items: center;
    width: 100%;
    max-width: 1000px;
    height: 100%;
}

/* Media Queries */
@media screen and (max-width: 768px) {
    #join-dialog {
        width: 90%;
        padding: 30px;
    }

    #join-dialog h2 {
        font-size: 50px;
    }

    #layout-container {
        grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
    }
}

@media screen and (max-width: 480px) {
    #join-dialog {
        width: 100%;
        padding: 20px;
    }

    #join-dialog h2 {
        font-size: 40px;
    }

    #layout-container {
        grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
    }
}
</style>
