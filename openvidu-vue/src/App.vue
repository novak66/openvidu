<script lang="ts">
import {
    LocalVideoTrack,
    createLocalTracks,
    RemoteTrack,
    RemoteTrackPublication,
    Room,
    RoomEvent,
    RemoteVideoTrack,
    LocalTrack,
    createLocalVideoTrack,
    createLocalAudioTrack,
    RemoteParticipant,
    Track
} from 'livekit-client';

import VideoComponent from './components/VideoComponent.vue';
import AudioComponent from './components/AudioComponent.vue';

type TrackInfo = {
    trackPublication: RemoteTrackPublication;
    participantIdentity: string;
    track: LocalTrack | RemoteTrack;
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
            fluxo: 0,
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
            entrou: false as Boolean,
            microfoneAtivo: false as boolean,
            cameraAtiva: false as boolean,
            alreadyConnected: false,
            newRoomName: '' as string,
            isScreenSharing: false
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

        window.addEventListener('beforeunload', this.leaveRoom());
    },

    beforeDestroy() {
        this.leaveRoom();
        window.removeEventListener('beforeunload', this.leaveRoom());
    },

    computed: {
        participantsMap() {
            const participants = new Map();

            this.remoteTracksMap.forEach((trackInfo) => {
                const identity = trackInfo.participantIdentity;

                if (!participants.has(identity)) {
                    participants.set(identity, {
                        identity: identity,
                        videoTrack: null,
                        audioTrack: null,
                        isCameraMuted: true,
                        isMicMuted: true
                    });
                }

                const participant = participants.get(identity);
                if (trackInfo.track.kind === 'video') {
                    participant.videoTrack = trackInfo.track;
                    participant.isCameraMuted = trackInfo.track.isMuted; // ✅ Verifica se está mutado
                } else if (trackInfo.track.kind === 'audio') {
                    participant.audioTrack = trackInfo.track;
                    participant.isMicMuted = trackInfo.track.isMuted; // ✅ Verifica se está mutado
                }
            });

            return participants;
        }
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
                        participantIdentity: participant.identity,
                        track: _track
                    });
                }
            );

            this.room.on(RoomEvent.Connected, () => {
                //console.log('✅ RoomEvent.Connected');
            });

            this.room.on(RoomEvent.Disconnected, (reason) => {
                //console.error('❌ RoomEvent.Disconnected:', reason);
            });

            this.room.on(RoomEvent.Reconnecting, () => {
                // console.warn('🔄 RoomEvent.Reconnecting');
            });

            this.room.on(RoomEvent.Reconnected, () => {
                // console.log('✅ RoomEvent.Reconnected');
            });

            this.room.on(RoomEvent.SignalConnected, () => {
                // console.log('✅ RoomEvent.SignalConnected - WebSocket conectado!');
            });

            this.room.on(RoomEvent.TrackSubscribed, (...args) => {
                //  console.log('✅ RoomEvent.TrackSubscribed:', args);
            });

            this.room.on(RoomEvent.ParticipantConnected, () => {
                //  console.log('moreParticipants');
                //this.connectingParticipants();
            });
            this.room.on(RoomEvent.ParticipantDisconnected, (participant: RemoteParticipant) => {
                this.remoteTracksMap.forEach((trackInfo, trackSid) => {
                    if (trackInfo.participantIdentity === participant.identity) {
                        this.remoteTracksMap.delete(trackSid);
                    }
                });
            });

            this.connectingParticipants();

            this.room.on(RoomEvent.Connected, () => {
                //  console.log('✅ RoomEvent.Connected');
            });
        },

        async toggleScreenShare() {
            try {
                if (!this.isScreenSharing) {
                    // Iniciar compartilhamento
                    await this.room?.localParticipant.setScreenShareEnabled(true);
                    this.isScreenSharing = true;
                    console.log('✅ Compartilhamento de tela iniciado!');
                } else {
                    // Parar compartilhamento
                    await this.room?.localParticipant.setScreenShareEnabled(false);
                    this.isScreenSharing = false;
                    console.log('❌ Compartilhamento de tela parado!');
                }
            } catch (error) {
                console.error('❌ Erro ao compartilhar tela:', error);
                alert('Não foi possível compartilhar a tela. Verifique as permissões do navegador.');
            }
        },

        async connectingParticipants() {
            if (this.alreadyConnected) {
                return;
            }

            try {
                const stream = await navigator.mediaDevices.getUserMedia({
                    video: true,
                    audio: true
                });
                stream.getTracks().forEach((track) => track.stop());
            } catch (error) {
                console.error('❌ Permissões negadas:', error);
            }

            try {
                this.localVideoTrack = await createLocalVideoTrack();
            } catch (error) {
                console.error('❌ Erro ao criar vídeo:', error);
            }

            try {
                this.localAudioTrack = await createLocalAudioTrack();
            } catch (error) {
                console.error('❌ Erro ao criar áudio:', error);
            }

            try {
                if (this.room) {
                    const token = await this.getToken(this.participantName);
                    await this.room.connect(this.LIVEKIT_URL, token);

                    if (this.localVideoTrack) {
                        await this.room.localParticipant.publishTrack(this.localVideoTrack);
                        this.localTrack = this.localVideoTrack;
                        this.cameraAtiva = true;
                    }

                    if (this.localAudioTrack) {
                        await this.room.localParticipant.publishTrack(this.localAudioTrack);
                        this.microfoneAtivo = true;
                    }

                    this.entrou = true;

                    this.room.localParticipant.trackPublications.forEach((pub, sid) => {
                        console.log(`  - ${pub.kind}: ${sid}`);
                    });
                }
            } catch (error: any) {
                console.error('❌ ERRO:', error);
                this.leaveRoom();
            }
            this.alreadyConnected = true;
        },

        async leaveRoom() {
            this.fluxo = 0;
            this.entrou = false;
            this.alreadyConnected = false;
            this.roomSelected = null;
            if (this.room) {
                await this.room.disconnect();
            }

            this.room = undefined;
            this.localTrack = undefined;
            this.remoteTracksMap.clear();
        },

        async mutarDesmutar() {
            if (!this.room) return;

            const audioTrack = this.room.localParticipant.getTrackPublication(Track.Source.Microphone);

            if (audioTrack && audioTrack.track) {
                if (audioTrack.isMuted) {
                    await audioTrack.track.unmute();

                    this.microfoneAtivo = true;
                } else {
                    await audioTrack.track.mute();

                    this.microfoneAtivo = false;
                }
            } else {
                console.warn('⚠️ Nenhuma track de áudio publicada');
            }
        },

        async changeCamera() {
            if (!this.localVideoTrack) {
                return;
            }

            if (this.cameraAtiva) {
                await this.localVideoTrack.mute();
                this.cameraAtiva = false;
            } else {
                await this.localVideoTrack.unmute();
                this.cameraAtiva = true;
            }
        },

        async config(): Promise<string> {
            const select = this.roomSelected;
            const response = await fetch(
                this.APPLICATION_OPENVIDU_V1_SERVER_URL + 'rooms/' + select?.roomId + '/config',
                {
                    method: 'GET',
                    headers: {
                        'Content-Type': 'application/json',
                        Authorization: 'Bearer ' + this.loginToken
                    }
                }
            );

            const data = await response.json();
            return data.token;
        },

        async getToken(participantName: string): Promise<string> {
            const select = this.roomSelected;

            const u = new URL(select?.moderatorUrl);
            const secret = u.searchParams.get('secret');
            console.log(secret);
            const response = await fetch(this.APPLICATION_OPENVIDU_INTERNAL_SERVER_URL + 'participants/token', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify({ roomId: select?.roomId, participantName, secret })
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

        async createRoom(): Promise<any> {
            const response = await fetch(this.APPLICATION_OPENVIDU_V1_SERVER_URL + 'rooms', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json', Authorization: 'Bearer ' + this.loginToken },
                body: JSON.stringify({ roomName: this.newRoomName })
            });

            if (!response.ok) {
                const error = await response.json();
                throw new Error(`Falha ao criar sala: ${error.errorMessage}`);
            }
            this.newRoomName = '';

            await response.json();
            this.rooms = await this.getRooms();
        },

        convertData(date: number) {
            const d = new Date(date);

            return d.toLocaleString('pt-BR', { timeZone: 'America/Sao_Paulo' });
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
        <div style="display: flex; flex-direction: column" v-if="fluxo == 0">
            <div style="display: flex; align-items: center; gap: 8px; margin-bottom: 24px">
                <input
                    v-model="newRoomName"
                    type="text"
                    placeholder="Nome da sala"
                    style="
                        padding: 8px 14px;
                        border: 1px solid #ccc;
                        border-radius: 10px;
                        font-size: 14px;
                        width: 240px;
                        transition: all 0.2s ease;
                        outline: none;
                    "
                    onfocus="this.style.borderColor='#6b73ff'; this.style.boxShadow='0 0 0 2px rgba(107,115,255,0.2)'"
                    onblur="this.style.borderColor='#ccc'; this.style.boxShadow='none'"
                />
                <button
                    style="
                        background: linear-gradient(90deg, #6b73ff, #8b63ff);
                        color: white;
                        font-weight: 600;
                        padding: 8px 18px;
                        border: none;
                        border-radius: 10px;
                        cursor: pointer;
                        transition: opacity 0.2s ease;
                    "
                    onmouseover="this.style.opacity='0.9'"
                    onmouseout="this.style.opacity='1'"
                    @click="createRoom()"
                >
                    Criar sala
                </button>
            </div>

            <div style="display: flex; justify-content: space-around; flex-wrap: wrap">
                <div
                    style="
                        width: 32%;
                        border: 1px solid rgba(0, 0, 0, 0.2);
                        display: flex;
                        justify-content: center;
                        margin-bottom: 10px;
                    "
                    v-for="r in rooms"
                >
                    <div
                        style="
                            width: 100%;
                            display: flex;
                            flex-direction: column;
                            justify-content: flex-start;
                            align-items: flex-start;
                            gap: 8px;
                        "
                    >
                        <div
                            style="
                                width: 100%;
                                display: flex;
                                flex-direction: column;
                                justify-content: flex-start;
                                align-items: flex-start;
                                background: #f7fafc;
                                padding: 30px;
                            "
                        >
                            <div
                                style="width: 100%; display: flex; justify-content: space-between; align-items: center"
                            >
                                <h1
                                    style="
                                        font-size: 18px;
                                        background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
                                        width: 40px;
                                        height: 40px;
                                        display: flex;
                                        justify-content: center;
                                        align-items: center;
                                        border-radius: 12px;
                                    "
                                >
                                    📹
                                </h1>
                                <div
                                    style="
                                        background: #d1fae5;
                                        color: #065f46;
                                        border-radius: 20px;
                                        width: 80px;
                                        height: 30px;
                                        display: flex;
                                        padding: 10px;
                                        justify-content: space-between;
                                        align-items: center;
                                    "
                                    v-if="r.status === 'open'"
                                >
                                    <span
                                        style="background: #10b981; width: 8px; height: 8px; border-radius: 50%"
                                    ></span>
                                    <div style="margin-bottom: 4px">Aberta</div>
                                </div>
                                <div
                                    v-else
                                    style="
                                        background: #42a5f5;
                                        color: #1976d2;
                                        border-radius: 20px;
                                        width: fit-content;
                                        height: 30px;
                                        display: flex;
                                        padding: 10px;
                                        gap: 10px;
                                        justify-content: space-between;
                                        align-items: center;
                                    "
                                >
                                    <span
                                        style="background: #1565c0; width: 8px; height: 8px; border-radius: 50%"
                                    ></span>
                                    <div style="margin-bottom: 4px">Reunião em andamento</div>
                                </div>
                            </div>
                            <div>
                                <h3>
                                    {{ r.roomName }}
                                </h3>
                            </div>
                        </div>
                        <div style="padding: 20px; color: #a0aec0; font-size: 13px">
                            {{ convertData(r.creationDate) }}
                        </div>
                        <div style="display: flex; justify-content: center; width: 100%">
                            <button
                                style="
                                    width: 80%;
                                    margin-bottom: 30px;
                                    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
                                    border: none;
                                    border-radius: 10px;
                                    font-weight: 600;
                                    cursor: pointer;
                                    padding: 16px;
                                    color: white;
                                "
                                @click="((roomSelected = r), (fluxo = 1))"
                            >
                                Entrar na sala
                            </button>
                        </div>
                    </div>
                </div>
            </div>
        </div>
        <div v-else>
            <div>
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
                        <button class="btn btn-danger" id="leave-room-button" @click="leaveRoom()">Leave Room</button>
                    </div>
                    <div id="layout-container">
                        <VideoComponent
                            :track="localTrack"
                            :participantIdentity="participantName"
                            :local="true"
                            :audio="microfoneAtivo"
                            :video="cameraAtiva"
                            @change-camera="changeCamera()"
                            @change-microphone="mutarDesmutar()"
                            @toggle-screen-share="toggleScreenShare()"
                        />
                        <template v-for="participant of participantsMap.values()" :key="participant.identity">
                            <VideoComponent
                                :track="participant.videoTrack"
                                :participantIdentity="participant.identity"
                            />
                            <AudioComponent v-if="participant.audioTrack" :track="participant.audioTrack" hidden />
                        </template>
                    </div>
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
