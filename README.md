<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Chat Log Visualizer</title>
    
<script src="https://cdn.tailwindcss.com"></script>
    
<link rel="stylesheet" as="style" crossorigin href="https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.9/dist/web/static/pretendard.min.css" />
    <style>
        body { font-family: 'Pretendard', sans-serif; }
        
        /* 내 말풍선 스타일 */
        .bubble-mine {
            position: relative;
            margin-right: 10px; 
            padding: 8px 12px; 
            display: inline-block; 
        }
        .bubble-mine::before { /* 실제 꼬리 */
            content: '';
            position: absolute;
            bottom: 0px;
            right: -7px; 
            width: 10px;
            height: 10px;
            background-color: theme('colors.pink.500'); 
            border-bottom-right-radius: 50%;
        }
        .bubble-mine::after { /* 배경 채우기 */
            content: '';
            position: absolute;
            bottom: 0px;
            right: -10px; 
            width: 10px;
            height: 10px;
            background-color: theme('colors.white'); 
            border-bottom-right-radius: 50%;
        }

        /* 상대방 말풍선 스타일 */
        .bubble-other {
            position: relative;
            margin-left: 10px; 
            padding: 8px 12px; 
            display: inline-block; 
        }
        .bubble-other::before { /* 실제 꼬리 */
            content: '';
            position: absolute;
            bottom: 0px;
            left: -7px; 
            width: 10px;
            height: 10px;
            background-color: theme('colors.gray.200'); 
            border-bottom-left-radius: 50%;
        }
        .bubble-other::after { /* 배경 채우기 */
            content: '';
            position: absolute;
            bottom: 0px;
            left: -10px; 
            width: 10px;
            height: 10px;
            background-color: theme('colors.white'); 
            border-bottom-left-radius: 50%;
        }
    </style>
</head>
<body class="bg-gray-50 flex items-center justify-center min-h-screen p-4">

    
<div id="app" class="w-full max-w-xl lg:max-w-3xl shadow-2xl bg-white rounded-3xl overflow-hidden min-h-[90vh] flex flex-col">
        
</div>

    <script type="module">
        // Import Firebase modules
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, addDoc, setDoc, doc, setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Firebase Globals (Mandatory for Canvas Environment)
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');

        // Initialize Firebase
        let app, db, auth, userId = null;
        
        try {
            if (Object.keys(firebaseConfig).length > 0) {
                app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                auth = getAuth(app);
                setLogLevel('Debug'); // Enable logging for debugging
            }
        } catch (error) {
            console.error("Firebase initialization failed:", error);
        }

        /**
         * Firebase Authentication Setup
         */
        async function authenticateUser() {
            if (!auth) return;
            try {
                // __initial_auth_token이 정의되어 있으면 Custom Token으로 로그인, 아니면 익명 로그인
                if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                    await signInWithCustomToken(auth, __initial_auth_token);
                } else {
                    await signInAnonymously(auth);
                }
                // 로그인 성공 후 userId 설정
                userId = auth.currentUser?.uid || crypto.randomUUID();
                console.log("Firebase Auth successful. User ID:", userId);
            } catch (error) {
                console.error("Firebase Auth failed:", error);
                userId = crypto.randomUUID(); // 인증 실패 시 임시 ID
            }
        }

        // 애플리케이션 상태 관리
        const appState = {
            currentPage: 'setup', 
            myRole: null, 
            nameA: localStorage.getItem('chatApp_nameA') || 'PL1', // 초기값을 PL1로 설정하고 로컬 스토리지에서 로드
            nameB: localStorage.getItem('chatApp_nameB') || 'PL2', // 초기값을 PL2로 설정하고 로컬 스토리지에서 로드
            imageA: null,
            imageB: null,
            rawChatText: '', // 파일 업로드로 읽어들인 텍스트
            chatLog: []
        };

        const appElement = document.getElementById('app');
        // P를 A 또는 B로 대체하여 플레이어 기본 아이콘으로 사용
        const defaultAvatar = 'https://placehold.co/40x40/f472b6/ffffff?text=P'; 
        
        
        // --- Utility Functions ---

        /**
         * 정규식 특수 문자를 이스케이프합니다. (이름에 특수 문자가 포함되어도 파싱 가능하도록)
         */
        function escapeRegExp(string) {
            return string.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
        }

        /**
         * Base64 이미지 및 이름 상태를 로컬 스토리지에서 로드합니다.
         */
        function loadState() {
            // 이름은 이미 appState 초기화 시 로드됨. 이미지 로드만 수행.
            const storedImageA = localStorage.getItem('chatApp_imageA');
            const storedImageB = localStorage.getItem('chatApp_imageB');
            
            if (storedImageA) appState.imageA = storedImageA;
            if (storedImageB) appState.imageB = storedImageB;
        }

        /**
         * 파일을 읽어 Base64로 변환합니다. (프로필 사진용)
         */
        function _handleImageUpload(file, role) {
            if (!file) return;

            const reader = new FileReader();
            reader.onload = function(e) {
                if (role === 'A') {
                    appState.imageA = e.target.result;
                    localStorage.setItem('chatApp_imageA', e.target.result);
                } else {
                    appState.imageB = e.target.result;
                    localStorage.setItem('chatApp_imageB', e.target.result);
                }
                render();
            };
            reader.readAsDataURL(file);
        }
        window.handleImageUpload = _handleImageUpload; // 전역 바인딩

        /**
         * 업로드된 채팅 로그 파일을 읽어 appState.rawChatText에 저장합니다.
         */
        function handleFileChange(file) {
            if (!file) {
                appState.rawChatText = '';
                // 파일 선택 취소 시 버튼 비활성화를 위해 렌더링
                render(); 
                return;
            }
            const reader = new FileReader();
            reader.onload = function(e) {
                appState.rawChatText = e.target.result;
                // 파일 내용 로드 후 버튼 상태 등을 업데이트하기 위해 render() 호출
                render(); 
            };
            reader.readAsText(file);
        }


        /**
         * 채팅 텍스트를 파싱하여 메시지 배열로 변환합니다. 타임스탬프 포함.
         * 사용자가 입력한 이름에 따라 동적으로 정규식을 생성합니다.
         */
        function parseChatLog() {
            const lines = appState.rawChatText.split('\n');
            const messages = [];

            // 사용자가 입력한 현재 이름들을 사용
            const nameA = appState.nameA.trim(); 
            const nameB = appState.nameB.trim(); 

            // 이름이 비어있으면 파싱 불가
            if (!nameA || !nameB) return [];

            // 1. 동적으로 파싱할 이름 목록 생성 및 정규식 문자열 이스케이프
            const escapedNameA = escapeRegExp(nameA);
            const escapedNameB = escapeRegExp(nameB);
            
            // 두 이름 중 하나에 매칭되는 패턴
            const senderPattern = `(${escapedNameA}|${escapedNameB})`;
            
            // 2. 동적 정규식 생성 (타임스탬프 형식은 유지)
            // 정규식: (날짜 및 시간)\s*(이름 패턴)\s*(메시지 내용)
            const dynamicChatLineRegex = new RegExp(
                `^(.*?(?:오전|오후)\\s\\d{1,2}:\\d{2})\\s*${senderPattern}\\s*(.*)`
            );


            lines.forEach(line => {
                const match = line.match(dynamicChatLineRegex);
                if (match) {
                    const timestamp = match[1].trim(); 
                    const senderNameRaw = match[2].trim();
                    const messageText = match[3].trimLeft().trimRight(); 

                    let senderRole = null;
                    let senderName = senderNameRaw;

                    // 4. 역할 할당 로직: nameA(Role A) 또는 nameB(Role B)에 매칭되는지 확인
                    if (senderNameRaw === nameA) {
                        senderRole = 'A';
                    } 
                    else if (senderNameRaw === nameB) {
                        senderRole = 'B';
                    }

                    if (senderRole && messageText) {
                        // 불필요한 메시지 필터링
                        if (!messageText.includes('스티커를 남겼습니다.') && !messageText.includes('사진을 보냈습니다.') && !messageText.includes('이모티콘을 보냈습니다.')) {
                            messages.push({
                                sender: senderRole,
                                text: messageText,
                                name: senderName, // 파싱된 이름 그대로 저장
                                timestamp: timestamp
                            });
                        }
                    }
                }
            });

            if (messages.length === 0 && appState.rawChatText.trim().length > 0) {
                 console.warn("채팅 로그 파싱 실패: 이름 설정과 파일 형식을 확인해주세요.");
            }

            return messages;
        }

        /**
         * '나'의 역할을 설정하고 채팅 화면으로 전환합니다.
         */
        function _startChat(role) {
            // 이름이 비어있는지 확인
            if (appState.nameA.trim() === '' || appState.nameB.trim() === '') {
                 showModal('오류', '두 플레이어의 이름을 모두 입력해주세요.');
                 return;
            }
            
            appState.chatLog = parseChatLog();
            
            // 파일이 업로드되지 않았거나 파싱된 메시지가 없는 경우
            if (appState.rawChatText.trim().length === 0) {
                showModal('오류', '채팅 로그 파일을 먼저 업로드해주세요.');
                return;
            }
            if (appState.chatLog.length === 0) {
                showModal('오류', '유효한 채팅 메시지를 찾을 수 없습니다. 이름 설정과 채팅 로그 형식을 확인해주세요.');
                return;
            }

            appState.myRole = role;
            appState.currentPage = 'chat';
            render();
        }
        window.startChat = _startChat; // 전역 바인딩

        /**
         * Firestore에 현재 채팅 로그를 저장합니다.
         */
        async function _saveChatLog() {
            if (!db || !userId) {
                showModal('저장 오류', '데이터베이스 연결 또는 사용자 인증에 문제가 있습니다.');
                return;
            }

            if (appState.chatLog.length === 0) {
                 showModal('저장 오류', '저장할 채팅 내용이 없습니다.');
                 return;
            }

            // 저장할 데이터
            const chatData = {
                createdAt: new Date().toISOString(),
                userId: userId,
                nameA: appState.nameA,
                nameB: appState.nameB,
                imageA: appState.imageA,
                imageB: appState.imageB,
                // chatLog 배열은 크기 제한으로 인해 JSON 문자열로 저장합니다.
                chatLogJson: JSON.stringify(appState.chatLog), 
                logLength: appState.chatLog.length
            };

            try {
                // Private data path: /artifacts/{appId}/users/{userId}/chat_logs
                const logsCollection = collection(db, `artifacts/${appId}/users/${userId}/chat_logs`);
                const docRef = await addDoc(logsCollection, chatData);
                
                showModal('저장 완료!', `채팅방이 성공적으로 저장되었습니다.\n\n나중에 다시 불러오기 위한 ID:\n${docRef.id}`);
            } catch (error) {
                console.error("채팅 로그 저장 중 오류 발생:", error);
                showModal('저장 실패', '채팅방 저장 중 오류가 발생했습니다. 콘솔을 확인해주세요.');
            }
        }
        window.saveChatLog = _saveChatLog; // 전역 바인딩

        /**
         * 커스텀 모달을 표시합니다.
         */
        function showModal(title, message) {
            // 기존 모달 제거
            document.getElementById('custom-modal')?.remove();

            const modalHtml = `
                <div id="custom-modal" class="fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center z-50">
                    <div class="bg-white p-6 rounded-xl shadow-2xl max-w-sm w-full">
                        <h3 class="text-xl font-bold mb-4 text-gray-800">${title}</h3>
                        <p class="text-gray-700 mb-6 whitespace-pre-wrap">${message}</p>
                        <button onclick="document.getElementById('custom-modal').remove()" 
                                class="w-full bg-pink-500 hover:bg-pink-600 text-white font-semibold py-2 rounded-lg transition duration-200">
                            확인
                        </button>
                    </div>
                </div>
            `;
            document.body.insertAdjacentHTML('beforeend', modalHtml);
        }
        window.showModal = showModal; // 전역 바인딩 (모달 내 버튼에서 호출될 수 있도록)

        // --- Component Rendering Functions ---

        /**
         * 프로필 사진 업로드 컴포넌트를 렌더링합니다.
         */
        function renderProfileUploader(role, name, image) {
            const currentImage = image || defaultAvatar.replace('P', role);
            const inputId = `image-upload-${role}`;

            // 플레이어 이름 입력 필드
            return `
                <div class="flex flex-col items-center p-4 bg-white rounded-xl border border-pink-100 shadow-md flex-1 min-w-[150px]">
                    <label for="${inputId}" class="cursor-pointer">
                        <img src="${currentImage}" class="w-20 h-20 rounded-full object-cover ring-4 ring-pink-300 hover:ring-pink-500 transition duration-300" alt="프로필 사진" onerror="this.onerror=null;this.src='${defaultAvatar.replace('P', role)}';">
                    </label>
                    <input type="file" id="${inputId}" class="hidden" accept="image/*" onchange="handleImageUpload(this.files[0], '${role}')">

                    <p class="mt-3 font-semibold text-lg text-gray-800">PL${role === 'A' ? '1' : '2'}</p>
                    <input type="text"
                           value="${name}"
                           oninput="updateName('${role}', this.value)"
                           placeholder="이름 입력"
                           class="mt-1 text-center text-sm font-medium text-pink-500 border-b border-pink-200 focus:outline-none focus:border-pink-500 transition duration-150">
                </div>
            `;
        }

        /**
         * 설정 화면을 렌더링합니다. (파일 업로드 버전)
         */
        function renderSetupScreen() {
            // 파일이 로드되었는지 확인하여 버튼 활성화/비활성화 상태를 결정할 수 있음
            const isFileLoaded = appState.rawChatText.trim().length > 0;
            const buttonClasses = isFileLoaded 
                ? 'bg-pink-500 hover:bg-pink-600 shadow-lg shadow-pink-200' 
                : 'bg-gray-300 cursor-not-allowed';

            return `
                <div class="p-6 md:p-10 flex flex-col gap-8">
                    <h1 class="text-3xl font-bold text-center text-gray-800 border-b pb-4">Chat Log Visualizer</h1>

                    
<div class="flex justify-around items-start gap-4 p-4 bg-pink-50 rounded-xl flex-wrap">
                        ${renderProfileUploader('A', appState.nameA, appState.imageA)}
                        <div class="text-3xl text-pink-400 mt-6 font-thin hidden lg:block">
                           <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-8 h-8">
                              <path stroke-linecap="round" stroke-linejoin="round" d="M21 8.25c0-2.485-2.099-4.5-4.688-4.5-1.935 0-3.597 1.004-4.368 2.332-.771-1.328-2.433-2.332-4.368-2.332C5.1 3.75 3 5.765 3 8.25c0 7.22 9 12 9 12s9-4.78 9-12Z" />
                           </svg>
                        </div>
                        ${renderProfileUploader('B', appState.nameB, appState.imageB)}
                    </div>

                    
<div class="flex flex-col items-center">
                        <label for="chat-file" class="text-lg font-extrabold text-gray-800 mb-2 text-center">채팅 로그 파일 업로드 (.txt)</label>
                        <input type="file" id="chat-file" accept=".txt" class="w-full max-w-sm p-4 border border-gray-300 rounded-xl focus:ring-2 focus:ring-pink-500 focus:border-pink-500 transition duration-150" autofocus>
                        <p class="text-sm text-gray-500 mt-1">
                           ${appState.rawChatText.trim().length > 0 ? `<span class="text-green-600 font-semibold">파일 로드 완료.</span> ${appState.rawChatText.split('\n').length}줄 감지됨.` : '파일을 선택해주세요.'}
                        </p>
                    </div>

                    
<div class="flex flex-col gap-3 pt-4 border-t border-gray-100">
                        <button onclick="startChat('A')" class="w-full text-white font-bold py-3 px-6 rounded-xl transition duration-200 ${buttonClasses}">
                            나를 PL1 (${appState.nameA})로 보기
                        </button>
                        <button onclick="startChat('B')" class="w-full bg-pink-200 hover:bg-pink-300 text-pink-800 font-bold py-3 px-6 rounded-xl transition duration-200 shadow-md shadow-pink-100">
                            나를 PL2 (${appState.nameB})로 보기
                        </button>
                    </div>
                </div>
            `;
        }

        /**
         * 채팅 화면을 렌더링합니다.
         */
        function renderChatScreen() {
            const myRole = appState.myRole;
            const myName = myRole === 'A' ? appState.nameA : appState.nameB;
            
            // 상대방 정보는 현재 선택된 역할과 반대되는 역할의 정보
            const otherRole = myRole === 'A' ? 'B' : 'A';
            const otherName = otherRole === 'A' ? appState.nameA : appState.nameB;

            // 메시지 맵핑 및 렌더링
            const messagesHtml = appState.chatLog.map((msg, index) => {
                // 메시지 발신자(msg.sender)가 현재 설정된 '나의 역할'(myRole)과 일치하는지 확인
                const isMine = msg.sender === myRole;

                // 해당 메시지 발신자의 이름과 아바타 정보를 가져옵니다.
                const nameToShow = msg.sender === 'A' ? appState.nameA : appState.nameB;
                const avatarSrc = msg.sender === 'A' 
                    ? (appState.imageA || defaultAvatar.replace('P', 'A')) 
                    : (appState.imageB || defaultAvatar.replace('P', 'B'));


                const bubbleClasses = isMine
                    ? 'bg-pink-500 text-white bubble-mine' // 내 메시지 스타일 (오른쪽)
                    : 'bg-gray-200 text-gray-800 bubble-other'; // 상대방 메시지 스타일 (왼쪽)

                // 상대방 메시지일 때만 프로필 및 이름을 표시 (화자가 바뀌거나 첫 메시지일 경우)
                // 내 메시지일 때는 프로필을 표시하지 않습니다.
                const showProfileAndName = !isMine && (index === 0 || appState.chatLog[index - 1].sender !== msg.sender);
                
                // --- Render Logic ---

                if (isMine) {
                    // My Message (Right-aligned)
                    return `
                        <div class="flex justify-end mb-3 px-4 w-full items-end">
                            
<div class="flex flex-col items-end max-w-[75%]">
                                
<div class="flex items-end">
                                    <div class="text-xs text-gray-400 mr-1 whitespace-nowrap">
                                        ${msg.timestamp}
                                    </div>
                                    <div class="text-sm rounded-2xl shadow-md ${bubbleClasses} whitespace-pre-wrap">
                                        ${msg.text}
                                    </div>
                                </div>
                            </div>
                        </div>
                    `;
                } else {
                    // Other's Message (Left-aligned with Avatar/Name)
                    const profileColumn = showProfileAndName ? `
                        <div class="flex flex-col items-center w-10">
                            <img src="${avatarSrc}" class="w-10 h-10 rounded-full object-cover shadow" alt="${nameToShow}">
                            <p class="text-xs text-gray-500 mt-1 truncate max-w-full font-medium">${nameToShow}</p>
                        </div>
                    ` : '<div class="w-10 h-10"></div>'; // Placeholder for alignment

                    return `
                        <div class="flex justify-start mb-3 px-4 w-full items-start">
                            
${profileColumn}

                            
<div class="flex flex-col max-w-[75%] ml-2">
                                
<div class="flex items-end">
                                    <div class="text-sm rounded-2xl shadow-md ${bubbleClasses} whitespace-pre-wrap">
                                        ${msg.text}
                                    </div>
                                    <div class="text-xs text-gray-400 ml-1 whitespace-nowrap">
                                        ${msg.timestamp}
                                    </div>
                                </div>
                            </div>
                        </div>
                    `;
                }
            }).join('');


            return `
                
<div class="p-4 bg-pink-50 border-b border-pink-200 flex items-center justify-between shadow-md">
                    <button onclick="goToSetup()" class="text-pink-600 hover:text-pink-800 transition duration-150">
                        <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor" class="w-6 h-6">
                            <path stroke-linecap="round" stroke-linejoin="round" d="M10.5 19.5 3 12m0 0 7.5-7.5M3 12h18" />
                        </svg>
                    </button>
                    
<h2 class="text-lg font-semibold text-gray-800 truncate">${myName} (${otherName}과 대화 중)</h2>
                    
                    <button onclick="saveChatLog()" class="bg-pink-500 hover:bg-pink-600 text-white font-semibold py-1 px-3 rounded-full text-sm transition duration-200 shadow-md">
                        저장하기
                    </button>
</div>

                
<div id="chat-window" class="flex-1 overflow-y-auto p-4 space-y-3 bg-white">
                    ${messagesHtml.length > 0 ? messagesHtml : '<p class="text-center text-gray-500 mt-10">채팅 메시지가 없습니다.</p>'}
                </div>

                
<div class="p-3 border-t border-gray-200 bg-gray-50 flex items-center">
                    <input type="text" placeholder="메시지 입력..." disabled class="flex-1 p-3 border border-gray-300 rounded-full bg-white text-gray-600 focus:outline-none focus:ring-2 focus:ring-pink-500">
                    <button disabled class="ml-2 bg-pink-300 text-white p-3 rounded-full opacity-50">
                         <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" class="w-5 h-5">
                            <path d="M3.105 2.289a.75.75 0 0 0-.826.807l.38 1.67a.75.75 0 0 0 .617.58l1.353.187 3.51-.555a.75.75 0 0 1 .58.26l4.23 6.345a.75.75 0 0 1-.58.125l-2.073-.171c-.742-.061-1.398.42-1.637 1.144l-.357 1.071a.75.75 0 0 0 1.05.974l2.56-1.536a.75.75 0 0 0 .285-.595V8.153a4.5 4.5 0 0 0-4.5-4.5H5.857a.75.75 0 0 1-.722-.593l-.38-1.67a.75.75 0 0 0-.74-.598Z" />
                        </svg>
                    </button>
                </div>
            `;
        }

        // --- Event Handlers (Global Scope Binding) ---

        function _updateName(role, value) {
            // 이름 업데이트 및 로컬 스토리지 저장
            if (role === 'A') {
                appState.nameA = value;
                localStorage.setItem('chatApp_nameA', value);
            } else {
                appState.nameB = value;
                localStorage.setItem('chatApp_nameB', value);
            }
            
            // 이름 입력 시 끊김 현상을 방지하기 위해 전체 render() 호출을 제거하고,
            // 이름이 반영되는 버튼 텍스트만 수동으로 업데이트합니다.
            const buttonA = document.querySelector('button[onclick="startChat(\'A\')"]');
            const buttonB = document.querySelector('button[onclick="startChat(\'B\')"]');

            if (buttonA) {
                buttonA.textContent = `나를 PL1 (${appState.nameA})로 보기`;
                // 파일 로드 상태에 따라 버튼 활성화/비활성화 클래스 유지
                if (appState.rawChatText.trim().length > 0) {
                     buttonA.classList.remove('bg-gray-300', 'cursor-not-allowed');
                     buttonA.classList.add('bg-pink-500', 'hover:bg-pink-600', 'shadow-lg', 'shadow-pink-200');
                }
            }
            if (buttonB) {
                buttonB.textContent = `나를 PL2 (${appState.nameB})로 보기`;
            }
        }
        window.updateName = _updateName; // 전역 바인딩

        function _goToSetup() {
            appState.currentPage = 'setup';
            appState.myRole = null;
            appState.chatLog = []; // 로그 초기화
            render();
        }
        window.goToSetup = _goToSetup; // 전역 바인딩

        /**
         * 메인 렌더링 함수
         */
        function render() {
            if (appState.currentPage === 'setup') {
                appElement.innerHTML = renderSetupScreen();
                
                // 파일 입력 이벤트 리스너 재설정
                const fileInput = document.getElementById('chat-file');
                if (fileInput) {
                    fileInput.onchange = (e) => {
                        handleFileChange(e.target.files[0]);
                    };
                }

            } else if (appState.currentPage === 'chat') {
                appElement.innerHTML = renderChatScreen();
                // 채팅창 맨 아래로 스크롤
                const chatWindow = document.getElementById('chat-window');
                if (chatWindow) {
                    chatWindow.scrollTop = chatWindow.scrollHeight;
                }
            }
        }

        // --- Initialization ---

        window.onload = async function() {
            await authenticateUser(); // Firebase 인증을 먼저 수행
            loadState();
            render();
        };
    </script>
</body>
</html>
