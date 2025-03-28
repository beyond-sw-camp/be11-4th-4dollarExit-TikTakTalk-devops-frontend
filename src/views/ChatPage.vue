<template>
    <v-container class="chat-container">
        <v-row class="chat-row">
            <!-- 채팅방 목록 -->
            <v-col cols="12" md="4" class="chat-list-column">
                <v-card class="chat-list-card" elevation="0">
                    <v-card-title class="text-center text-h6">
                        내 라운지
                        <div class="total-unread-count">
                            읽지 않은 메시지 {{ totalUnreadCount }}개
                        </div>
                    </v-card-title>
                    <v-card-text>
                        <div class="chat-rooms">
                            <v-list>
                                <v-list-item
                                    v-for="chat in chatList"
                                    :key="chat.roomId"
                                    :class="{'active-chat': chat.roomId === roomId}"
                                    @click="enterChatRoom(chat.roomId)"
                                >
                                    <div class="d-flex justify-space-between align-center w-100">
                                        <div class="room-info">
                                            <div class="room-name-container">
                                                <span>{{ getChatRoomName(senderNickName, chat.roomName) }}</span>
                                                <span v-if="chat.isGroupChat === 'Y'" class="participant-count">
                                                    ({{ chat.chatPaticipantCount }})
                                                </span>
                                                <span v-if="chat.unReadCount > 0" class="unread-indicator">
                                                    {{ chat.unReadCount }}
                                                </span>
                                            </div>
                                        </div>
                                        <v-btn
                                            v-if="chat.isGroupChat === 'Y'"
                                            density="compact"
                                            icon="mdi-exit-to-app"
                                            variant="text"
                                            color="error"
                                            @click.stop="openLeaveModal(chat.roomId)"
                                        ></v-btn>
                                    </div>
                                </v-list-item>
                            </v-list>
                        </div>
                    </v-card-text>
                </v-card>
            </v-col>

            <!-- 채팅창 -->
            <v-col cols="12" md="7" class="chat-main-column">
                <v-card class="chat-card" elevation="0">
                    <v-card-title class="chat-room-header">
                        {{ getChatRoomName(senderNickName, currentChatRoom ? currentChatRoom.roomName : '라운지를 선택해주세요!') }}
                    </v-card-title>
                    <div class="chat-content-wrapper">
                        <div class="chat-box" ref="chatBox">
                            <template v-for="(msg, index) in messages" :key="`msg-group-${index}`">
                                <!-- 날짜 구분선 -->
                                <div 
                                    v-if="shouldShowDateDivider(msg, messages[index-1])"
                                    class="date-divider"
                                >
                                    <span>{{ formatDate(msg.sendTime) }}</span>
                                </div>
                                
                                <!-- 채팅 메시지 -->
                                <div 
                                    :class="['chat-message', msg.senderNickName === senderNickName ? 'sent' : 'received']"
                                >
                                    <!-- 받은 메시지일 때만 프로필 이미지 표시 -->
                                    <div v-if="msg.senderNickName !== senderNickName" class="profile-image">
                                        <img 
                                            :src="msg.senderImagePath || require('@/assets/basicProfileImage.png')"
                                            :alt="msg.senderNickName"
                                            class="rounded-square"
                                        >
                                    </div>
                                    <div class="message-content">
                                        <strong v-if="msg.senderNickName !== senderNickName">{{ msg.senderNickName }}</strong>
                                        <p class="message-text">{{ msg.message }}</p>
                                        <span class="message-time">{{ formatTime(msg.sendTime) }}</span>
                                    </div>
                                </div>
                            </template>
                        </div>
                        <div class="input-area">
                            <v-text-field
                                v-model="newMessage"
                                @keyup.enter="sendMessage"
                                variant="outlined"
                                density="comfortable"
                                hide-details
                                class="message-input"
                                placeholder="메시지를 입력하세요..."
                                :bg-color="'white'"
                                label=""
                            />
                            <v-btn color="primary" block @click="sendMessage" class="send-button" elevation="0">
                                전송
                            </v-btn>
                        </div>
                    </div>
                </v-card>
            </v-col>
        </v-row>

        <!-- 채팅방 나가기 확인 모달 -->
        <v-dialog v-model="showLeaveModal" max-width="400">
            <v-card class="leave-modal">
                <v-card-text class="leave-modal-content">
                    <div class="leave-icon-wrapper">
                        <v-icon class="leave-icon" color="error" size="32">mdi-exit-to-app</v-icon>
                    </div>
                    <h3 class="leave-title">채팅방 나가기</h3>
                    <p class="leave-description">
                        정말로 나갈거에요? 🥺<br>
                        다른 사람들이 슬퍼할 거예요 😢
                    </p>
                </v-card-text>
                <v-card-actions class="leave-actions">
                    <div class="button-wrapper">
                        <v-btn
                            variant="outlined"
                            color="grey-darken-1"
                            class="cancel-button"
                            @click="showLeaveModal = false"
                        >
                            취소
                        </v-btn>
                        <v-btn
                            color="error"
                            class="leave-button"
                            @click="confirmLeave"
                            :loading="isLeaving"
                        >
                            나가기
                        </v-btn>
                    </div>
                </v-card-actions>
            </v-card>
        </v-dialog>
    </v-container>
    <div v-if="!isConnected" class="reconnect-notice">
    서버와 연결이 끊겼습니다. 재연결 시도 중...
    </div>
</template>

<script>
import Stomp from 'webstomp-client';
import axios from 'axios';
import SockJS from 'sockjs-client';

export default{
    data(){
        return {
            messages: [],
            newMessage: "",
            stompClient: null,
            token: "",
            senderNickName: null,
            roomId: null,
            currentChatRoom: null,
            showLeaveModal: false,
            tempRoomId: null,
            isLeaving: false,
            // 파드 오토스케일링에 따른 문제점 보완
            reconnectInterval: null, //재연결 시도 주기 저장
            isConnected: false, // 연결 상태 체크
            isManuallyDisconnected: false, // 사용자의 의도인지 체크
        }
    },
    computed: {
        chatList() {
            return this.$store.state.chatList;
        },
        totalUnreadCount() {
            return this.chatList.reduce((total, chat) => total + (chat.unReadCount || 0), 0);
        }
    },
    async created(){
        this.senderNickName = localStorage.getItem("nickName");
        this.roomId = this.$route.params.roomId;
        
        // 채팅방 목록 먼저 가져오기
        const chatListResponse = await axios.get(`${process.env.VUE_APP_API_BASE_URL}/chat/my/rooms`);
        
        if (this.roomId) {
            // 채팅방 목록에서 현재 채팅방 정보 찾기
            const chatList = chatListResponse.data.result;
            this.currentChatRoom = chatList.find(chat => chat.roomId === Number(this.roomId));
            
            // 채팅 히스토리 가져오기
            const response = await axios.get(`${process.env.VUE_APP_API_BASE_URL}/chat/history/${this.roomId}`);
            this.messages = response.data.result;
            this.connectWebsocket();
            
            // 채팅 히스토리를 불러온 후 nextTick을 사용하여 DOM 업데이트 후 스크롤
            this.$nextTick(() => {
                this.scrollToBottom();
            });
        }
        
        // 채팅방 목록 저장
        this.$store.dispatch('setChatList', chatListResponse.data.result.sort((a, b) => b.unReadCount - a.unReadCount));
    },
    // 사용자가 현재 라우트에서 다른 라우트로 이동하려고 할때 호출되는 훅함수
    beforeRouteLeave(to, from, next) {
        this.disconnectWebSocket();
        next();
    },
    // 화면을 완전히 꺼버렸을때
    beforeUnmount() {
        this.disconnectWebSocket();
    },
    methods: {
        connectWebsocket() {
            if (this.stompClient && this.stompClient.connected) return;

            const sockJs = new SockJS(`https://server.tiktaktok.site/connect`);
            this.stompClient = Stomp.over(sockJs);
            this.token = localStorage.getItem("token");

            this.stompClient.connect(
                { Authorization: `Bearer ${this.token}` },
                () => {
                    console.log("소켓 연결");
                    this.isConnected = true;
                    this.isManuallyDisconnected = false; // 성공 시 수동 종료 해제

                    this.stompClient.subscribe(
                        `/topic/${this.roomId}`,
                        (message) => {
                            const parseMessage = JSON.parse(message.body);
                            this.messages.push(parseMessage);
                            this.scrollToBottom();
                        },
                        { Authorization: `Bearer ${this.token}` }
                    );

                    // 연결 성공했으니까 재연결 루프 멈추기
                    if (this.reconnectInterval) {
                        clearInterval(this.reconnectInterval);
                        this.reconnectInterval = null;
                    }
                },
                (error) => {
                    console.error("소켓 연결 실패:", error);
                    this.isConnected = false;
                    if (!this.isManuallyDisconnected) {
                        this.scheduleReconnect(); // 사용자가 끊은게 아니면 재연결
                    }
                }
            );

            sockJs.onclose = () => {
                console.warn("소켓 연결 종료됨. 재연결 시도 예정");
                this.isConnected = false;
                if (!this.isManuallyDisconnected) {
                    this.scheduleReconnect(); // 사용자가 끊은게 아니면 재연결
                }
            };
        },
        scheduleReconnect() {
            if (!this.reconnectInterval) {
                this.reconnectInterval = setInterval(() => {
                    console.log(" 소켓 재연결 시도");
                    this.connectWebsocket();
                }, 5000); // 5초마다 재연결
            }
        },
        sendMessage() {
            if (this.newMessage.trim() === "") return;

            const message = {
                senderNickName: this.senderNickName,
                message: this.newMessage
            };

            console.log("📤 전송할 메시지:", message);

            if (this.stompClient && this.stompClient.connected) {
                console.log("✅ STOMP 연결 상태 확인됨");
                this.stompClient.send(`/publish/${this.roomId}`, JSON.stringify(message));
                console.log("✅ 메시지 전송 성공!");
            } else {
                console.error("❌ STOMP 클라이언트가 연결되지 않음");
            }

            this.newMessage = "";
            this.scrollToBottom();
        },
        scrollToBottom() {
            this.$nextTick(() => {
                if (this.$refs.chatBox) {
                    this.$refs.chatBox.scrollTop = this.$refs.chatBox.scrollHeight;
                }
            });
        },
        async disconnectWebSocket() {
            if (this.stompClient && this.stompClient.connected) {
                this.stompClient.disconnect();
                this.isConnected = false;
            }
            // 사용자가 직접 끊은거로 표시
            this.isManuallyDisconnected = true;

            // 재연결 루프 멈추기
            if (this.reconnectInterval) {
                clearInterval(this.reconnectInterval);
                this.reconnectInterval = null;
            }

            // 읽음 처리
            if (this.roomId && this.currentChatRoom) {
                try {
                    await axios.post(`${process.env.VUE_APP_API_BASE_URL}/chat/room/${this.roomId}/read`);
                } catch (error) {
                    console.log('읽음 처리 실패:', error);
                }
            }
        },
        async enterChatRoom(roomId) {
            if (this.roomId !== roomId) {
                try {
                    // 기존 웹소켓 연결이 있다면 먼저 끊기
                    if (this.roomId) {
                        this.disconnectWebSocket();
                    }
                    
                    this.roomId = roomId;
                    this.currentChatRoom = this.chatList.find(chat => chat.roomId === roomId);
                    
                    // 새로운 채팅방의 히스토리 가져오기
                    const response = await axios.get(`${process.env.VUE_APP_API_BASE_URL}/chat/history/${roomId}`);
                    this.messages = response.data.result;
                    
                    // 읽지 않은 메시지 처리
                    await axios.post(`${process.env.VUE_APP_API_BASE_URL}/chat/room/${roomId}/read`);
                    // chatList에서 해당 채팅방의 unReadCount를 0으로 업데이트
                    if (this.currentChatRoom) {
                        this.currentChatRoom.unReadCount = 0;
                    }
                    
                    // 새로운 웹소켓 연결
                    this.connectWebsocket();
                    this.scrollToBottom();
                } catch (error) {
                    console.error('채팅방 입장 실패:', error);
                }
            }
        },
        openLeaveModal(roomId) {
            this.tempRoomId = roomId;
            this.showLeaveModal = true;
        },
        async confirmLeave() {
            try {
                this.isLeaving = true;
                await this.leaveChatRoom(this.tempRoomId);
                this.showLeaveModal = false;
            } catch (error) {
                console.error('채팅방 나가기 실패:', error);
            } finally {
                this.isLeaving = false;
                this.tempRoomId = null;
            }
        },
        async leaveChatRoom(roomId) {
            try {
                // 소켓 끊기 (정상 종료로 표시됨)
                if (this.roomId === roomId) {
                    await this.disconnectWebSocket();
                }

                // 나가기 API
                await axios.delete(`${process.env.VUE_APP_API_BASE_URL}/chat/room/group/${roomId}/leave`);

                // 채팅방 목록 갱신
                await this.$store.dispatch('removeChatRoom', roomId);
                const response = await axios.get(`${process.env.VUE_APP_API_BASE_URL}/chat/my/rooms`);
                this.$store.dispatch('setChatList', response.data.result);

                // 현재 방 나가면 다른 방으로 이동
                if (this.roomId === roomId) {
                    if (this.chatList.length > 0) {
                        this.enterChatRoom(this.chatList[0].roomId);
                    } else {
                        this.roomId = null;
                        this.currentChatRoom = null;
                        this.messages = [];
                    }
                }
            } catch (error) {
                console.error('채팅방 나가기 실패:', error);
            }
        },
        getChatRoomName(currentUser, roomName) {
            if (roomName.includes('-')) {
                const [user1, user2] = roomName.split('-');
                
                return currentUser === user1 ? user2 : user1;
            }
            
            return roomName;
        },
        formatTime(timestamp) {
            if (!timestamp) return '';
            const date = new Date(timestamp);
            const hours = date.getHours();
            const minutes = date.getMinutes();
            const ampm = hours >= 12 ? '오후' : '오전';
            const formattedHours = hours % 12 || 12;
            const formattedMinutes = minutes < 10 ? `0${minutes}` : minutes;
            
            return `${ampm} ${formattedHours}:${formattedMinutes}`;
        },
        shouldShowDateDivider(currentMsg, prevMsg) {
            if (!currentMsg || !prevMsg) return false;
            const currentDate = new Date(currentMsg.sendTime).toDateString();
            const prevDate = new Date(prevMsg.sendTime).toDateString();
            return currentDate !== prevDate;
        },
        formatDate(timestamp) {
            if (!timestamp) return '';
            const date = new Date(timestamp);
            return date.toLocaleDateString();
        },
    },
    mounted() {
        this.scrollToBottom();
    },
    updated() {
        this.scrollToBottom();
    }
}
</script>
<style scoped>
.chat-container {
    max-width: 1200px !important;
    margin: 0 auto;
    padding: 40px;
    background: #F8FAFC;
    min-height: 100vh;
}

.chat-row {
    gap: 24px;
    background: white;
    border-radius: 24px;
    box-shadow: 0 4px 24px rgba(37, 99, 235, 0.06);
    max-width: 1100px;
    margin: 0 auto;
}

.chat-list-column {
    padding: 24px;
    border-right: 1px solid rgba(37, 99, 235, 0.08);
    position: relative;
}

.chat-list-column::after {
    content: '';
    position: absolute;
    right: 0;
    top: 5%;
    height: 90%;
    width: 1px;
    background: linear-gradient(
        to bottom,
        transparent,
        rgba(37, 99, 235, 0.15),
        transparent
    );
}

.chat-main-column {
    padding: 24px;
    background: #FAFBFF;
    border-top-right-radius: 24px;
    border-bottom-right-radius: 24px;
}

.chat-card, .chat-list-card {
    height: calc(100vh - 160px);
    background: transparent;
    box-shadow: none;
}

.chat-list-header {
    padding: 16px 20px;
    border-bottom: 1px solid rgba(0, 0, 0, 0.12);
    font-weight: 600;
}

.chat-rooms {
    padding: 0;
    height: calc(100vh - 160px);
    overflow-y: auto;
    padding-bottom: 40px;
}

.v-list {
    padding: 0 !important;
    padding-bottom: 40px !important;
    background: transparent;
}

.chat-room-item {
    min-height: 44px !important;  /* 높이 조정 */
    padding: 10px 16px !important;  /* 상하 패딩 줄임 */
    margin: 0 !important;  /* 마진 완전 제거 */
    border-radius: 0 !important;  /* 라운드 제거 */
    border-bottom: 1px solid rgba(0, 0, 0, 0.05);  /* 얇은 구분선 */
    transition: all 0.2s ease;
}

.chat-room-item:hover {
    background-color: #F4F1FA;
}

.active-chat {
    background-color: #F4F1FA !important;
    border-left: 3px solid #9155FD;
}

.chat-room-name {
    font-size: 0.95rem;
    color: #333;
    line-height: 1.2;  /* 줄 높이 조정 */
}

.unread-count {
    color: #9155FD;
    font-size: 0.85rem;
    margin-left: 4px;
}

/* Vuetify 기본 스타일 오버라이드 */
:deep(.v-list-item__content) {
    padding: 0 !important;
}

:deep(.v-list-item__title) {
    font-size: inherit !important;
}

.chat-message {
    display: flex;
    align-items: flex-start;
    margin: 12px 0;
    gap: 8px;
    margin-bottom: 24px;
}

.chat-message.sent {
    flex-direction: row-reverse;
}

.profile-image {
    flex-shrink: 0;
    width: 48px;
    height: 48px;
    margin-right: 4px;
    margin-top: 4px;
}

.profile-image img {
    width: 100%;
    height: 100%;
    border-radius: 50%;
    object-fit: cover;
    border: 1px solid rgba(0, 0, 0, 0.08);
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.message-content {
    display: flex;
    flex-direction: column;
    gap: 4px;
    position: relative;
}

.message-content strong {
    font-size: 0.9rem;
    color: #64748B;
}

.message-text {
    padding: 8px 12px;
    border-radius: 12px;
    max-width: 400px;
    word-break: break-word;
}

.sent .message-text {
    background-color: #2563EB;
    color: white;
    border-radius: 16px 0 16px 16px;
}

.received .message-text {
    background-color: #F1F5F9;
    border-radius: 0 16px 16px 16px;
}

.message-time {
    font-size: 0.75rem;
    color: #94A3B8;
    position: absolute;
    bottom: -18px;
    white-space: nowrap;
}

.sent .message-time {
    right: 0;
}

.received .message-time {
    left: 0;
}

.input-area {
    position: sticky;
    bottom: 0;
    padding: 20px;
    background: white;
    border-top: 1px solid rgba(37, 99, 235, 0.08);
    border-radius: 0 0 24px 24px;
    box-shadow: 0 -4px 16px rgba(37, 99, 235, 0.03);
}

.message-input {
    margin-bottom: 16px;
}

.message-input :deep(.v-field) {
    border-radius: 16px !important;
    background-color: #F8FAFC;
    border: 1px solid transparent;
    transition: all 0.2s ease;
}

.message-input :deep(.v-field--focused) {
    border-color: #2563EB !important;
    background-color: white;
    box-shadow: 0 4px 16px rgba(37, 99, 235, 0.08);
}

.send-button {
    height: 48px;
    font-size: 1rem;
    font-weight: 600;
    letter-spacing: 0.5px;
    background-color: #2563EB !important;
    border-radius: 16px;
    transition: all 0.2s ease;
    text-transform: none;
}

.send-button:hover {
    background-color: #1D4ED8 !important;
    transform: translateY(-2px);
    box-shadow: 0 6px 16px rgba(37, 99, 235, 0.2);
}

.chat-box {
    height: calc(100vh - 280px);
    overflow-y: auto;
    padding: 24px;
    background-color: white;
    border-radius: 16px;
    margin-bottom: 20px;
    box-shadow: 0 4px 16px rgba(37, 99, 235, 0.03);
}

.chat-box::-webkit-scrollbar {
    width: 6px;
}

.chat-box::-webkit-scrollbar-track {
    background: transparent;
}

.chat-box::-webkit-scrollbar-thumb {
    background: #CBD5E1;
    border-radius: 3px;
}

.chat-room-header {
    text-align: center;
    padding: 20px 0;
    font-weight: 600;
    font-size: 1.2rem;
    color: #1E293B;
    border-bottom: 1px solid rgba(37, 99, 235, 0.08);
    margin-bottom: 16px;
}

.chat-content-wrapper {
    position: relative;
    height: calc(100vh - 200px);
    display: flex;
    flex-direction: column;
}

/* 나가기 모달 스타일 */
.leave-modal {
    border-radius: 24px;
    overflow: hidden;
    background: white;
    box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.25);
}

.leave-modal-content {
    padding: 32px 32px 24px;
    text-align: center;
}

.leave-icon-wrapper {
    width: 64px;
    height: 64px;
    background: rgba(239, 68, 68, 0.1);
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    margin: 0 auto 20px;
    animation: pulse 2s infinite;
}

.leave-icon {
    animation: shake 0.5s ease-in-out;
}

.leave-title {
    font-size: 1.5rem;
    font-weight: 700;
    color: #1E293B;
    margin-bottom: 16px;
}

.leave-description {
    color: #64748B;
    line-height: 1.6;
    font-size: 0.95rem;
    margin: 0;
}

.leave-actions {
    padding: 16px 32px 32px;
    display: flex;
    justify-content: center;
}

.button-wrapper {
    display: flex;
    gap: 12px;
    justify-content: center;
    width: 100%;
}

.cancel-button, .leave-button {
    min-width: 120px;
    height: 44px;
    font-size: 0.95rem;
    font-weight: 600;
    text-transform: none;
    border-radius: 12px;
    letter-spacing: 0.3px;
}

.cancel-button {
    background-color: #F8FAFC !important;
}

.leave-button {
    background: linear-gradient(to right, #EF4444, #DC2626) !important;
    position: relative;
    overflow: hidden;
}

.leave-button::after {
    content: '';
    position: absolute;
    top: 0;
    left: -100%;
    width: 200%;
    height: 100%;
    background: linear-gradient(
        to right,
        transparent,
        rgba(255, 255, 255, 0.2),
        transparent
    );
    transition: 0.5s;
}

.leave-button:hover::after {
    left: 100%;
}

.leave-button:hover {
    transform: translateY(-2px);
    box-shadow: 0 8px 16px rgba(239, 68, 68, 0.2);
}

@keyframes shake {
    0%, 100% { transform: translateX(0); }
    25% { transform: translateX(-4px); }
    75% { transform: translateX(4px); }
}

@keyframes pulse {
    0% { transform: scale(1); opacity: 1; }
    50% { transform: scale(1.05); opacity: 0.8; }
    100% { transform: scale(1); opacity: 1; }
}

.date-divider {
    text-align: center;
    margin: 20px 0;
    position: relative;
    z-index: 1;
}

.date-divider::before {
    content: '';
    position: absolute;
    top: 50%;
    left: 0;
    right: 0;
    height: 1px;
    background-color: rgba(0, 0, 0, 0.08);
    z-index: -1;
}

.date-divider span {
    background-color: #F8FAFC;
    padding: 4px 16px;
    border-radius: 12px;
    font-size: 0.8rem;
    color: #64748B;
    border: 1px solid rgba(0, 0, 0, 0.08);
}

.chat-rooms::-webkit-scrollbar {
    width: 4px;
}

.chat-rooms::-webkit-scrollbar-track {
    background: transparent;
}

.chat-rooms::-webkit-scrollbar-thumb {
    background: #E2E8F0;
    border-radius: 4px;
    transition: background 0.2s ease;
}

.chat-rooms::-webkit-scrollbar-thumb:hover {
    background: #9155FD;
}

.chat-rooms::-webkit-scrollbar-thumb:active {
    background: #7934F3;
}

.reconnect-notice {
  position: fixed;
  top: 10px;
  left: 50%;
  transform: translateX(-50%);
  background: red;
  color: white;
  padding: 8px 16px;
  border-radius: 8px;
  z-index: 999;
}

.room-info {
    flex-grow: 1;
}

.room-name-container {
    display: flex;
    align-items: center;
    gap: 4px;
}

.room-name-container span:first-child {
    font-weight: 500;
    color: #1E293B;
}

.participant-count {
    color: #94A3B8;  /* 더 연한 회색으로 변경 */
    font-size: 0.85rem;  /* 글자 크기도 살짝 줄임 */
    font-weight: normal;
    opacity: 0.8;  /* 투명도 추가 */
}

.unread-indicator {
    background: linear-gradient(135deg, #9155FD, #7934F3);
    color: white;
    font-size: 0.7rem;
    padding: 2px 6px;
    border-radius: 8px;
    margin-left: 8px;
    font-weight: 500;
    min-width: 18px;
    height: 18px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    box-shadow: 0 2px 8px rgba(145, 85, 253, 0.25);
    animation: scaleIn 0.3s ease-out, pulse 2s infinite;
}

@keyframes scaleIn {
    from {
        transform: scale(0.8);
        opacity: 0;
    }
    to {
        transform: scale(1);
        opacity: 1;
    }
}

@keyframes pulse {
    0% {
        box-shadow: 0 2px 8px rgba(145, 85, 253, 0.25);
    }
    50% {
        box-shadow: 0 2px 12px rgba(145, 85, 253, 0.4);
    }
    100% {
        box-shadow: 0 2px 8px rgba(145, 85, 253, 0.25);
    }
}

.total-unread-count {
    font-size: 0.85rem;
    color: #64748B;
    font-weight: normal;
    margin-top: 4px;
}
</style>