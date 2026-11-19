# 4k-hunting<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Apex Horizon 3D - Realistic Hunting Simulator</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <link href="https://fonts.googleapis.com/css2?family=Chakra+Petch:ital,wght@0,400;0,600;0,700;1,700&family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
    <style>
        * {
            box-sizing: border-box;
            user-select: none;
            -webkit-user-select: none;
        }
        body, html {
            margin: 0;
            padding: 0;
            width: 100%;
            height: 100%;
            overflow: hidden;
            background-color: #0b0f14;
            font-family: 'Inter', sans-serif;
            color: #f3f4f6;
        }
        .font-tactical {
            font-family: 'Chakra Petch', sans-serif;
        }
        #gameCanvas {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            display: block;
            z-index: 1;
        }
        /* Telescopic Scope Overlay */
        #scopeOverlay {
            position: absolute;
            inset: 0;
            pointer-events: none;
            z-index: 10;
            display: none;
            overflow: hidden;
        }
        .scope-lens {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            width: min(84vw, 84vh);
            height: min(84vw, 84vh);
            border-radius: 50%;
            box-shadow: 0 0 0 2000px rgba(4, 7, 10, 0.96), inset 0 0 55px rgba(0, 0, 0, 0.95), inset 0 0 12px rgba(255, 255, 255, 0.15);
            border: 4px solid #1f2937;
        }
        /* Custom Scrollbar for Armory/Levels */
        ::-webkit-scrollbar {
            width: 6px;
        }
        ::-webkit-scrollbar-track {
            background: rgba(15, 23, 42, 0.6);
        }
        ::-webkit-scrollbar-thumb {
            background: #475569;
            border-radius: 3px;
        }
        ::-webkit-scrollbar-thumb:hover {
            background: #f59e0b;
        }
        @keyframes pulseDanger {
            0%, 100% { box-shadow: inset 0 0 30px rgba(220, 38, 38, 0.0); }
            50% { box-shadow: inset 0 0 90px rgba(220, 38, 38, 0.65); }
        }
        .danger-vignette {
            animation: pulseDanger 0.8s infinite;
        }
        @keyframes floatUp {
            0% { transform: translateY(0) scale(0.95); opacity: 1; }
            100% { transform: translateY(-45px) scale(1.05); opacity: 0; }
        }
        .hit-popup {
            animation: floatUp 1.8s cubic-bezier(0.16, 1, 0.3, 1) forwards;
        }
    </style>
</head>
<body class="relative w-full h-full overflow-hidden">

    <!-- 3D WebGL Canvas -->
    <canvas id="gameCanvas"></canvas>

    <!-- Predator Charge Danger Vignette -->
    <div id="dangerOverlay" class="fixed inset-0 pointer-events-none z-20 hidden danger-vignette"></div>

    <!-- Telescopic Rifle Scope Overlay -->
    <div id="scopeOverlay">
        <div class="scope-lens">
            <!-- Mil-Dot Reticle SVG -->
            <svg viewBox="0 0 400 400" class="w-full h-full">
                <!-- Outer Thick Posts -->
                <line x1="0" y1="200" x2="115" y2="200" stroke="#0a0a0a" stroke-width="5" />
                <line x1="285" y1="200" x2="400" y2="200" stroke="#0a0a0a" stroke-width="5" />
                <line x1="200" y1="285" x2="200" y2="400" stroke="#0a0a0a" stroke-width="5" />
                <line x1="200" y1="0" x2="200" y2="95" stroke="#0a0a0a" stroke-width="4" />

                <!-- Fine Crosshairs -->
                <line x1="115" y1="200" x2="285" y2="200" stroke="#111827" stroke-width="1.2" opacity="0.9" />
                <line x1="200" y1="95" x2="200" y2="285" stroke="#111827" stroke-width="1.2" opacity="0.9" />

                <!-- Illuminated Center Dot -->
                <circle cx="200" cy="200" r="2.2" fill="#ef4444" />
                <circle cx="200" cy="200" r="4.5" fill="none" stroke="#ef4444" stroke-width="0.6" opacity="0.7" />

                <!-- Horizontal Windage Mil-Dots -->
                <circle cx="140" cy="200" r="1.5" fill="#111" />
                <circle cx="160" cy="200" r="1.5" fill="#111" />
                <circle cx="180" cy="200" r="1.5" fill="#111" />
                <circle cx="220" cy="200" r="1.5" fill="#111" />
                <circle cx="240" cy="200" r="1.5" fill="#111" />
                <circle cx="260" cy="200" r="1.5" fill="#111" />

                <!-- Vertical Bullet-Drop Compensation Hash Marks -->
                <line x1="194" y1="220" x2="206" y2="220" stroke="#111" stroke-width="1.2" />
                <line x1="192" y1="240" x2="208" y2="240" stroke="#111" stroke-width="1.2" />
                <line x1="190" y1="260" x2="210" y2="260" stroke="#111" stroke-width="1.2" />
                <line x1="195" y1="180" x2="205" y2="180" stroke="#111" stroke-width="1.2" />
            </svg>

            <!-- Rangefinder & Ballistic Digital Readout inside Scope -->
            <div class="absolute top-[22%] left-1/2 -translate-x-1/2 flex items-center gap-3 bg-black/65 px-3 py-1 rounded border border-emerald-500/40 font-tactical text-xs sm:text-sm text-emerald-400 tracking-wider">
                <span>RNG: <strong id="scopeRangeVal">---</strong>m</span>
                <span class="text-slate-500">|</span>
                <span>ZOOM: <strong id="scopeZoomVal">4.0</strong>x</span>
            </div>

            <!-- Hold Breath Stamina Arc / Bar inside Scope -->
            <div class="absolute bottom-[20%] left-1/2 -translate-x-1/2 w-36 sm:w-44 flex flex-col items-center">
                <div class="text-[10px] font-tactical uppercase tracking-widest text-amber-300/90 mb-1" id="breathStatusText">HOLD [SHIFT] STEADY</div>
                <div class="w-full h-1.5 bg-slate-900/90 rounded-full overflow-hidden border border-slate-700">
                    <div id="breathBar" class="h-full bg-emerald-400 transition-all duration-75" style="width: 100%"></div>
                </div>
            </div>
        </div>
    </div>

    <!-- Unscoped Hip-Fire Crosshair -->
    <div id="hipCrosshair" class="fixed top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 pointer-events-none z-10 flex items-center justify-center">
        <div class="w-6 h-6 relative flex items-center justify-center opacity-75">
            <div class="w-1 h-1 bg-white rounded-full shadow"></div>
            <div class="absolute top-0 w-0.5 h-1.5 bg-white/80"></div>
            <div class="absolute bottom-0 w-0.5 h-1.5 bg-white/80"></div>
            <div class="absolute left-0 h-0.5 w-1.5 bg-white/80"></div>
            <div class="absolute right-0 h-0.5 w-1.5 bg-white/80"></div>
        </div>
    </div>

    <!-- Floating Anatomical / Ballistic Feedback Container -->
    <div id="feedbackContainer" class="fixed inset-0 pointer-events-none z-30 flex items-center justify-center"></div>

    <!-- IN-GAME TACTICAL HUD -->
    <div id="gameHud" class="fixed inset-0 pointer-events-none z-20 flex flex-col justify-between p-3 sm:p-5 hidden">
        <!-- Top Bar -->
        <div class="flex flex-wrap items-start justify-between gap-2">
            <!-- World & Level Progress -->
            <div class="bg-slate-950/80 backdrop-blur-md border border-slate-700/70 rounded-xl px-3.5 py-2.5 shadow-xl pointer-events-auto">
                <div class="flex items-center gap-2">
                    <span id="hudWorldBadge" class="px-2 py-0.5 rounded text-[11px] font-tactical font-bold uppercase tracking-wider bg-amber-500/20 text-amber-400 border border-amber-500/40">WORLD 1</span>
                    <span id="hudWorldName" class="text-xs sm:text-sm font-semibold text-slate-200">Whispering Pine Forest</span>
                </div>
                <div class="mt-1 flex items-center gap-3">
                    <div class="font-tactical text-lg sm:text-xl font-bold text-white tracking-wide">
                        LEVEL <span id="hudLevelNum" class="text-amber-400">1</span><span class="text-slate-500 text-sm">/20</span>
                    </div>
                    <div class="h-4 w-[1px] bg-slate-700"></div>
                    <div class="text-xs sm:text-sm text-slate-300">
                        Target: <strong id="hudTargetText" class="text-emerald-400">1x Whitetail Deer</strong>
                    </div>
                </div>
            </div>

            <!-- Wind & Ballistics Meter -->
            <div class="bg-slate-950/80 backdrop-blur-md border border-slate-700/70 rounded-xl px-3.5 py-2 shadow-xl flex items-center gap-3">
                <div class="relative w-9 h-9 rounded-full bg-slate-900 border border-slate-600 flex items-center justify-center">
                    <svg id="windArrowSvg" viewBox="0 0 24 24" class="w-5 h-5 text-cyan-400 transition-transform duration-300">
                        <path fill="currentColor" d="M12 2L4.5 20.29l.71.71L12 18l6.79 3 .71-.71z"/>
                    </svg>
                </div>
                <div>
                    <div class="text-[10px] font-tactical uppercase tracking-wider text-slate-400">Crosswind</div>
                    <div class="font-tactical text-sm sm:text-base font-bold text-cyan-300">
                        <span id="hudWindSpeed">4.2</span> <span class="text-xs font-normal text-slate-400">m/s</span>
                        <span id="hudWindDirText" class="text-xs text-amber-400 ml-1">E</span>
                    </div>
                </div>
            </div>

            <!-- Cash, XP & Menu Button -->
            <div class="flex items-center gap-2 pointer-events-auto">
                <div class="bg-slate-950/80 backdrop-blur-md border border-slate-700/70 rounded-xl px-3.5 py-2 shadow-xl text-right">
                    <div class="font-tactical text-sm sm:text-base font-bold text-emerald-400">$<span id="hudCash">500</span></div>
                    <div class="text-[11px] text-slate-400 font-medium">XP: <span id="hudXp" class="text-slate-200">0</span></div>
                </div>
                <button id="btnPauseMenu" class="bg-slate-900/90 hover:bg-slate-800 border border-slate-600 text-slate-200 rounded-xl px-3.5 py-2.5 text-xs sm:text-sm font-tactical font-bold tracking-wider transition shadow-lg cursor-pointer">
                    CAMP / MENU
                </button>
            </div>
        </div>

        <!-- Center Bottom Notification Banner -->
        <div id="hudAlertBanner" class="self-center bg-slate-950/85 border border-amber-500/50 text-amber-300 px-4 py-1.5 rounded-full text-xs sm:text-sm font-medium shadow-lg opacity-0 transition-opacity duration-300">
            Scan the terrain and spot your target.
        </div>

        <!-- Bottom Bar: Weapon Status & Mobile/Desktop Action Controls -->
        <div class="flex flex-wrap items-end justify-between gap-3">
            <!-- Controls Hint / Touch Buttons -->
            <div class="flex flex-wrap items-center gap-2 pointer-events-auto">
                <button id="btnToggleScope" class="bg-slate-900/90 hover:bg-amber-600/30 border border-slate-600 hover:border-amber-400 text-slate-100 rounded-xl px-3.5 py-2.5 text-xs font-tactical font-bold tracking-wider transition shadow-lg cursor-pointer flex items-center gap-1.5">
                    <span>🔭</span> SCOPE <span class="hidden sm:inline text-slate-400">[RMB]</span>
                </button>
                <button id="btnHoldBreath" class="bg-slate-900/90 hover:bg-emerald-600/30 border border-slate-600 hover:border-emerald-400 text-slate-100 rounded-xl px-3.5 py-2.5 text-xs font-tactical font-bold tracking-wider transition shadow-lg cursor-pointer flex items-center gap-1.5">
                    <span>🫁</span> STEADY <span class="hidden sm:inline text-slate-400">[SHIFT]</span>
                </button>
                <button id="btnZoomCycle" class="bg-slate-900/90 hover:bg-cyan-600/30 border border-slate-600 hover:border-cyan-400 text-slate-100 rounded-xl px-3.5 py-2.5 text-xs font-tactical font-bold tracking-wider transition shadow-lg cursor-pointer">
                    🔍 ZOOM <span class="hidden sm:inline text-slate-400">[SCROLL/Z]</span>
                </button>
                <button id="btnReload" class="bg-slate-900/90 hover:bg-slate-800 border border-slate-600 text-slate-100 rounded-xl px-3.5 py-2.5 text-xs font-tactical font-bold tracking-wider transition shadow-lg cursor-pointer">
                    🔄 RELOAD <span class="hidden sm:inline text-slate-400">[R]</span>
                </button>
                <button id="btnMobileFire" class="sm:hidden bg-red-600 active:bg-red-700 border border-red-400 text-white rounded-xl px-5 py-2.5 text-xs font-tactical font-bold tracking-wider shadow-lg cursor-pointer">
                    🔥 FIRE
                </button>
            </div>

            <!-- Active Rifle & Ammo Display -->
            <div class="bg-slate-950/85 backdrop-blur-md border border-slate-700/80 rounded-xl px-4 py-2.5 shadow-xl flex items-center gap-4 pointer-events-auto">
                <div>
                    <div id="hudGunName" class="font-tactical text-xs sm:text-sm font-bold text-amber-400 tracking-wide uppercase">Bolt-Action .243 Rifle</div>
                    <div class="text-[11px] text-slate-400" id="hudGunSpecs">Power: 45 | Zoom: 4x-8x</div>
                </div>
                <div class="h-8 w-[1px] bg-slate-700"></div>
                <div class="text-right">
                    <div id="hudAmmoDisplay" class="font-tactical text-xl sm:text-2xl font-bold text-white leading-none">4 <span class="text-xs text-slate-400">/ 4</span></div>
                    <div id="hudReloadStatus" class="text-[10px] font-tactical uppercase tracking-widest text-emerald-400 mt-0.5">CHAMBERED</div>
                </div>
            </div>
        </div>
    </div>

    <!-- MAIN BASE CAMP / MENU MODAL (Worlds, Levels 1-20, Armory) -->
    <div id="mainMenuOverlay" class="fixed inset-0 z-40 bg-slate-950/85 backdrop-blur-md flex items-center justify-center p-3 sm:p-6">
        <div class="w-full max-w-5xl max-h-[92vh] bg-slate-900/95 border border-slate-700/80 rounded-2xl shadow-2xl flex flex-col overflow-hidden">
            
            <!-- Header -->
            <div class="px-5 py-4 bg-gradient-to-r from-slate-950 via-slate-900 to-slate-950 border-b border-slate-800 flex flex-wrap items-center justify-between gap-3">
                <div class="flex items-center gap-3">
                    <div class="w-10 h-10 rounded-xl bg-amber-500/20 border border-amber-500/40 flex items-center justify-center text-amber-400 text-xl font-bold">
                        🎯
                    </div>
                    <div>
                        <h1 class="font-tactical text-xl sm:text-2xl font-bold tracking-wider text-white uppercase">APEX HORIZON <span class="text-amber-400">3D</span></h1>
                        <p class="text-xs text-slate-400">Tactical Ballistics & Big Game Hunting Expeditions</p>
                    </div>
                </div>

                <div class="flex items-center gap-3">
                    <div class="bg-slate-950 border border-slate-800 rounded-xl px-3.5 py-1.5 text-right">
                        <div class="text-[10px] text-slate-400 uppercase font-tactical">Hunter Funds</div>
                        <div class="font-tactical text-base sm:text-lg font-bold text-emerald-400">$<span id="menuCash">500</span></div>
                    </div>
                    <div class="bg-slate-950 border border-slate-800 rounded-xl px-3.5 py-1.5 text-right">
                        <div class="text-[10px] text-slate-400 uppercase font-tactical">Rank XP</div>
                        <div class="font-tactical text-base sm:text-lg font-bold text-amber-400"><span id="menuXp">0</span></div>
                    </div>
                </div>
            </div>

            <!-- Navigation Tabs -->
            <div class="flex border-b border-slate-800 bg-slate-950/60 px-4 pt-2 gap-2">
                <button id="tabExpeditions" class="px-4 py-2.5 font-tactical text-xs sm:text-sm font-bold uppercase tracking-wider rounded-t-lg border-t border-x border-slate-700 bg-slate-900 text-amber-400 cursor-pointer">
                    🌍 Worlds & Levels (1-20)
                </button>
                <button id="tabArmory" class="px-4 py-2.5 font-tactical text-xs sm:text-sm font-bold uppercase tracking-wider rounded-t-lg border-t border-x border-transparent text-slate-400 hover:text-slate-200 cursor-pointer">
                    🔫 Armory & Upgrades
                </button>
                <button id="tabBallistics" class="px-4 py-2.5 font-tactical text-xs sm:text-sm font-bold uppercase tracking-wider rounded-t-lg border-t border-x border-transparent text-slate-400 hover:text-slate-200 cursor-pointer">
                    📖 Field Guide
                </button>
            </div>

            <!-- Tab Content Area -->
            <div class="flex-1 overflow-y-auto p-4 sm:p-6">
                
                <!-- TAB 1: WORLDS & LEVELS -->
                <div id="contentExpeditions" class="space-y-5">
                    <!-- World Selector Cards -->
                    <div>
                        <h2 class="font-tactical text-xs uppercase tracking-widest text-slate-400 mb-2.5">Select Hunting Region (Beat Level 20 to Unlock Next World)</h2>
                        <div id="worldCardsGrid" class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-3">
                            <!-- Populated by JS -->
                        </div>
                    </div>

                    <!-- Selected World Level Grid (1 to 20) -->
                    <div class="bg-slate-950/70 border border-slate-800 rounded-xl p-4">
                        <div class="flex flex-wrap items-center justify-between gap-2 mb-3">
                            <div>
                                <h3 id="selectedWorldTitle" class="font-tactical text-lg font-bold text-white">World 1: Whispering Pine Forest</h3>
                                <p id="selectedWorldDesc" class="text-xs text-slate-400">Lush North American conifers. Native wildlife: Whitetail Deer, Wild Boar, Bull Elk, Grizzly Bear.</p>
                            </div>
                            <button id="btnContinueNextLevel" class="bg-amber-500 hover:bg-amber-400 text-slate-950 font-tactical font-bold text-xs sm:text-sm px-4 py-2.5 rounded-xl shadow-lg transition cursor-pointer">
                                ▶ PLAY NEXT UNLOCKED LEVEL
                            </button>
                        </div>

                        <div id="levelButtonsGrid" class="grid grid-cols-4 sm:grid-cols-5 md:grid-cols-10 gap-2 mt-3">
                            <!-- Populated by JS (Levels 1-20) -->
                        </div>
                    </div>
                </div>

                <!-- TAB 2: ARMORY & UPGRADES -->
                <div id="contentArmory" class="hidden space-y-4">
                    <div class="flex flex-wrap items-center justify-between gap-2">
                        <div>
                            <h2 class="font-tactical text-base sm:text-lg font-bold text-white">Rifle Rack & Ballistic Upgrades</h2>
                            <p class="text-xs text-slate-400">Equip higher-caliber rifles for large game & predators. Upgrade optics, stability, reload speed, and stopping power.</p>
                        </div>
                    </div>
                    <div id="armoryGrid" class="grid grid-cols-1 md:grid-cols-2 gap-4">
                        <!-- Populated by JS -->
                    </div>
                </div>

                <!-- TAB 3: FIELD GUIDE -->
                <div id="contentBallistics" class="hidden space-y-4 text-sm text-slate-300">
                    <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
                        <div class="bg-slate-950/80 border border-slate-800 rounded-xl p-4 space-y-2">
                            <h3 class="font-tactical text-amber-400 font-bold uppercase">🎯 Controls & Aiming</h3>
                            <ul class="space-y-1.5 text-xs sm:text-sm text-slate-300">
                                <li>• <strong>Aim / Pan:</strong> Move Mouse (or Drag on Touchscreens).</li>
                                <li>• <strong>Telescopic Scope:</strong> Right-Click (or press <kbd class="px-1.5 py-0.5 bg-slate-800 rounded text-amber-300">E</kbd> / Scope button).</li>
                                <li>• <strong>Variable Zoom:</strong> Mouse Wheel while scoped (or press <kbd class="px-1.5 py-0.5 bg-slate-800 rounded text-amber-300">Z</kbd>).</li>
                                <li>• <strong>Hold Breath:</strong> Hold <kbd class="px-1.5 py-0.5 bg-slate-800 rounded text-amber-300">Shift</kbd> or <kbd class="px-1.5 py-0.5 bg-slate-800 rounded text-amber-300">Space</kbd> to steady crosshair sway.</li>
                                <li>• <strong>Fire / Reload:</strong> Left-Click to fire, <kbd class="px-1.5 py-0.5 bg-slate-800 rounded text-amber-300">R</kbd> to reload magazine.</li>
                            </ul>
                        </div>
                        <div class="bg-slate-950/80 border border-slate-800 rounded-xl p-4 space-y-2">
                            <h3 class="font-tactical text-emerald-400 font-bold uppercase">🫀 Ballistics & Anatomical Hitboxes</h3>
                            <ul class="space-y-1.5 text-xs sm:text-sm text-slate-300">
                                <li>• <strong>Brain / Headshot (2.5x DMG):</strong> Instant takedown + <strong>+$75 Precision Bonus</strong>.</li>
                                <li>• <strong>Heart / Lungs Vital Shot (2.0x DMG):</strong> Clean ethical harvest + <strong>+$60 Vital Bonus</strong>.</li>
                                <li>• <strong>Body / Limb Shot (1.0x DMG):</strong> Wounds target; prey will sprint away and predators may charge!</li>
                                <li>• <strong>Gravity & Wind:</strong> Over 100m+, aim slightly above target using mil-dots and compensate for crosswind arrows.</li>
                            </ul>
                        </div>
                    </div>
                </div>

            </div>

            <!-- Footer -->
            <div class="px-5 py-3 bg-slate-950 border-t border-slate-800 flex items-center justify-between text-xs text-slate-500">
                <span>Progress auto-saves locally in your browser.</span>
                <button id="btnResetProgress" class="text-slate-500 hover:text-red-400 transition cursor-pointer">Reset Save Data</button>
            </div>
        </div>
    </div>

    <!-- LEVEL COMPLETE / FAILED SUMMARY MODAL -->
    <div id="roundEndModal" class="fixed inset-0 z-50 bg-slate-950/85 backdrop-blur-md hidden items-center justify-center p-4">
        <div class="w-full max-w-md bg-slate-900 border border-slate-700 rounded-2xl p-6 shadow-2xl text-center space-y-4">
            <div id="roundEndBadge" class="inline-block px-3 py-1 rounded-full text-xs font-tactical font-bold uppercase tracking-widest bg-emerald-500/20 text-emerald-400 border border-emerald-500/40">
                EXPEDITION SUCCESSFUL
            </div>
            <h2 id="roundEndTitle" class="font-tactical text-2xl sm:text-3xl font-bold text-white">LEVEL 1 COMPLETE</h2>
            <p id="roundEndSubtitle" class="text-xs sm:text-sm text-slate-400">Clean harvest recorded.</p>

            <!-- Rewards Breakdown -->
            <div id="roundRewardsBox" class="bg-slate-950/90 border border-slate-800 rounded-xl p-4 space-y-2 text-left text-xs sm:text-sm">
                <div class="flex justify-between">
                    <span class="text-slate-400">Base Bounty:</span>
                    <span class="font-tactical font-bold text-slate-200">+$<span id="rewBase">100</span></span>
                </div>
                <div class="flex justify-between">
                    <span class="text-slate-400">Vital / Headshot Bonus:</span>
                    <span class="font-tactical font-bold text-amber-400">+$<span id="rewVital">60</span></span>
                </div>
                <div class="flex justify-between">
                    <span class="text-slate-400">Long-Range & Stealth Bonus:</span>
                    <span class="font-tactical font-bold text-cyan-400">+$<span id="rewDist">40</span></span>
                </div>
                <div class="border-t border-slate-800 pt-2 flex justify-between text-base">
                    <span class="font-semibold text-white">Total Earned:</span>
                    <span class="font-tactical font-bold text-emerald-400">+$<span id="rewTotal">200</span></span>
                </div>
            </div>

            <div class="flex flex-col sm:flex-row gap-2.5 pt-2">
                <button id="btnEndBackCamp" class="flex-1 bg-slate-800 hover:bg-slate-700 text-slate-200 font-tactical font-bold text-xs sm:text-sm py-3 rounded-xl border border-slate-600 transition cursor-pointer">
                    CAMP & ARMORY
                </button>
                <button id="btnEndNextAction" class="flex-1 bg-amber-500 hover:bg-amber-400 text-slate-950 font-tactical font-bold text-xs sm:text-sm py-3 rounded-xl shadow-lg transition cursor-pointer">
                    NEXT LEVEL ▶
                </button>
            </div>
        </div>
    </div>

<script>
/**
 * ============================================================================
 * APEX HORIZON 3D - REALISTIC FIRST-PERSON HUNTING SIMULATOR
 * Complete Single-File Architecture:
 * 1. Game Data (4 Worlds x 20 Levels, 16 Unique Animals, 6 Rifles + Upgrades)
 * 2. Web Audio Procedural Sound Engine (Gunshots, Echo, Heartbeat, Wind, Hit)
 * 3. Three.js 3D Biome Renderer (Terrain, Trees, Weather Particles, Lighting)
 * 4. Articulated 3D Animal Builder & Hitbox System (Head, Heart/Lungs, Body)
 * 5. Ballistics Engine (Gravity Drop, Crosswind Drift, Bullet Tracer, X-Ray Popups)
 * 6. Dynamic AI State Machine (Graze, Patrol, Alert, Flee, Predator Charge)
 * ============================================================================
 */

// ----------------------------------------------------------------------------
// 1. WORLDS, SPECIES & ARMORY CONFIGURATION
// ----------------------------------------------------------------------------
const WORLDS = [
    {
        id: 0,
        name: "Whispering Pine Forest",
        region: "North America",
        desc: "Dense alpine conifers and misty meadows. Home to Whitetail Deer, Wild Boar, Bull Elk, and Grizzly Bears.",
        skyColor: 0x87a8bd,
        fogColor: 0x9bb7c7,
        fogDensity: 0.0042,
        groundColor: 0x3b5335,
        groundAccent: 0x4d6b42,
        treeType: "pine",
        weather: "mist",
        sunColor: 0xfff3d6,
        animals: ["deer", "boar", "elk", "grizzly"]
    },
    {
        id: 1,
        name: "Golden Savanna",
        region: "East Africa",
        desc: "Sun-drenched acacia plains and dusty ridges. Track Thomson's Gazelle, Plains Zebra, Cape Buffalo, and Lions.",
        skyColor: 0xe6b87d,
        fogColor: 0xdfb278,
        fogDensity: 0.0035,
        groundColor: 0xa68946,
        groundAccent: 0xc2a155,
        treeType: "acacia",
        weather: "dust",
        sunColor: 0xffe4a8,
        animals: ["gazelle", "zebra", "buffalo", "lion"]
    },
    {
        id: 2,
        name: "Frozen Tundra",
        region: "High Arctic",
        desc: "Blinding snowfields and jagged ice ridges. Hunt Arctic Fox, Barren-Ground Caribou, Musk Ox, and Polar Bears.",
        skyColor: 0xc8d9e6,
        fogColor: 0xd9e6f0,
        fogDensity: 0.0048,
        groundColor: 0xe6eef5,
        groundAccent: 0xc5d5e3,
        treeType: "snowpine",
        weather: "snow",
        sunColor: 0xeef7ff,
        animals: ["arctic_fox", "caribou", "musk_ox", "polar_bear"]
    },
    {
        id: 3,
        name: "Misty Bamboo Highlands",
        region: "Himalayan Asia",
        desc: "Towering bamboo groves and rugged cliffs. Stalk Sika Deer, Wild Yak, Snow Leopards, and Siberian Tigers.",
        skyColor: 0x6f8f86,
        fogColor: 0x7d9c93,
        fogDensity: 0.0052,
        groundColor: 0x2f4d3b,
        groundAccent: 0x3f634d,
        treeType: "bamboo",
        weather: "mist",
        sunColor: 0xdff5ec,
        animals: ["sika_deer", "wild_yak", "snow_leopard", "tiger"]
    }
];

const SPECIES_DB = {
    // World 1: Pine Forest
    deer: { name: "Whitetail Deer", hp: 90, speed: 13, scale: 1.0, bodyColor: 0x8c5a32, accentColor: 0xd9c5b2, antlers: "deer", predator: false, reward: 110, xp: 40 },
    boar: { name: "Wild Boar", hp: 115, speed: 12, scale: 0.85, bodyColor: 0x3d332c, accentColor: 0x6e5d52, antlers: "tusks", predator: false, reward: 130, xp: 50 },
    elk: { name: "Rocky Mountain Elk", hp: 155, speed: 13.5, scale: 1.28, bodyColor: 0x7a5638, accentColor: 0x3d2817, antlers: "elk", predator: false, reward: 165, xp: 65 },
    grizzly: { name: "Grizzly Bear", hp: 230, speed: 14.5, scale: 1.35, bodyColor: 0x4a3222, accentColor: 0x6e4d37, antlers: "none", predator: true, reward: 240, xp: 95 },

    // World 2: Golden Savanna
    gazelle: { name: "Thomson's Gazelle", hp: 85, speed: 16.5, scale: 0.88, bodyColor: 0xc28a49, accentColor: 0xf5efe6, antlers: "horns", predator: false, reward: 180, xp: 70 },
    zebra: { name: "Plains Zebra", hp: 145, speed: 15, scale: 1.15, bodyColor: 0xdedede, accentColor: 0x1c1c1c, antlers: "none", predator: false, reward: 210, xp: 85 },
    buffalo: { name: "Cape Buffalo", hp: 250, speed: 13.5, scale: 1.35, bodyColor: 0x262322, accentColor: 0x524c4a, antlers: "buffalo", predator: true, reward: 270, xp: 110 },
    lion: { name: "African Lion", hp: 240, speed: 16.5, scale: 1.25, bodyColor: 0xc79652, accentColor: 0x6e471e, antlers: "mane", predator: true, reward: 340, xp: 140 },

    // World 3: Frozen Tundra
    arctic_fox: { name: "Arctic Fox", hp: 75, speed: 16, scale: 0.68, bodyColor: 0xf2f6fa, accentColor: 0xcbd5e1, antlers: "none", predator: false, reward: 250, xp: 100 },
    caribou: { name: "Arctic Caribou", hp: 165, speed: 14.5, scale: 1.22, bodyColor: 0x7d746d, accentColor: 0xebe6e1, antlers: "elk", predator: false, reward: 290, xp: 120 },
    musk_ox: { name: "Musk Ox", hp: 280, speed: 12.5, scale: 1.35, bodyColor: 0x332924, accentColor: 0x75665e, antlers: "buffalo", predator: false, reward: 350, xp: 145 },
    polar_bear: { name: "Polar Bear", hp: 310, speed: 15.5, scale: 1.45, bodyColor: 0xf4f7f9, accentColor: 0xd1dce3, antlers: "none", predator: true, reward: 440, xp: 180 },

    // World 4: Misty Bamboo Highlands
    sika_deer: { name: "Himalayan Sika Deer", hp: 110, speed: 15, scale: 1.0, bodyColor: 0x8f562e, accentColor: 0xf0d9b5, antlers: "deer", predator: false, reward: 340, xp: 140 },
    wild_yak: { name: "Highland Wild Yak", hp: 300, speed: 13, scale: 1.4, bodyColor: 0x24201e, accentColor: 0x5c534e, antlers: "buffalo", predator: false, reward: 400, xp: 165 },
    snow_leopard: { name: "Snow Leopard", hp: 220, speed: 17.5, scale: 1.12, bodyColor: 0xcfd6db, accentColor: 0x4b5563, antlers: "none", predator: true, reward: 480, xp: 200 },
    tiger: { name: "Siberian Tiger", hp: 350, speed: 17.5, scale: 1.42, bodyColor: 0xd96b27, accentColor: 0x171717, antlers: "none", predator: true, reward: 600, xp: 250 }
};

const GUNS_DB = [
    {
        id: "rifle_243",
        name: "Bolt-Action .243 Rifle",
        caliber: ".243 Winchester",
        desc: "Lightweight starter bolt-action rifle. Accurate on small-to-medium game.",
        cost: 0,
        basePower: 55,
        baseZoomMin: 3.0,
        baseZoomMax: 6.0,
        baseStability: 0.55,
        baseReload: 2.1,
        magSize: 4,
        muzzleVel: 340
    },
    {
        id: "lever_3030",
        name: "Lever-Action .30-30",
        caliber: ".30-30 Repeater",
        desc: "Classic brush gun with fast follow-up shots and higher stopping power.",
        cost: 450,
        basePower: 80,
        baseZoomMin: 3.5,
        baseZoomMax: 7.5,
        baseStability: 0.62,
        baseReload: 1.8,
        magSize: 6,
        muzzleVel: 360
    },
    {
        id: "semi_308",
        name: "Semi-Auto .308 Marksman",
        caliber: "7.62x51mm NATO",
        desc: "Precision tactical semi-automatic rifle ideal for moving herds.",
        cost: 1100,
        basePower: 110,
        baseZoomMin: 4.0,
        baseZoomMax: 10.0,
        baseStability: 0.72,
        baseReload: 1.6,
        magSize: 8,
        muzzleVel: 410
    },
    {
        id: "magnum_300",
        name: ".300 Winchester Magnum",
        caliber: ".300 Win Mag",
        desc: "Flat-shooting long-range magnum built for Elk, Caribou, and Grizzlies.",
        cost: 2200,
        basePower: 150,
        baseZoomMin: 5.0,
        baseZoomMax: 12.0,
        baseStability: 0.78,
        baseReload: 1.9,
        magSize: 5,
        muzzleVel: 460
    },
    {
        id: "safari_375",
        name: ".375 H&H Safari Express",
        caliber: ".375 Holland & Holland",
        desc: "Heavy dangerous-game rifle engineered to stop charging Lions and Buffalo.",
        cost: 3800,
        basePower: 205,
        baseZoomMin: 5.0,
        baseZoomMax: 14.0,
        baseStability: 0.84,
        baseReload: 1.7,
        magSize: 4,
        muzzleVel: 490
    },
    {
        id: "bmg_50",
        name: ".50 BMG Anti-Materiel Rifle",
        caliber: "12.7x99mm BMG",
        desc: "Extreme ballistic devastation capable of single-shot vital takedowns on any apex beast.",
        cost: 6200,
        basePower: 310,
        baseZoomMin: 6.0,
        baseZoomMax: 16.0,
        baseStability: 0.90,
        baseReload: 2.2,
        magSize: 5,
        muzzleVel: 560
    }
];

// Generate Level Parameters (Levels 1 to 20 per world)
function getLevelConfig(worldIdx, levelNum) {
    const world = WORLDS[worldIdx];
    const speciesList = world.animals;
    let targetSpeciesKey = speciesList[0];
    let targetCount = 1;
    let ambientCount = 2;

    if (levelNum <= 4) {
        targetSpeciesKey = speciesList[0];
        targetCount = levelNum <= 2 ? 1 : 2;
    } else if (levelNum <= 9) {
        targetSpeciesKey = speciesList[1];
        targetCount = levelNum <= 7 ? 1 : 2;
    } else if (levelNum <= 14) {
        targetSpeciesKey = speciesList[2];
        targetCount = levelNum <= 12 ? 1 : 2;
    } else if (levelNum <= 19) {
        targetSpeciesKey = speciesList[3];
        targetCount = levelNum <= 17 ? 1 : 2;
    } else {
        // Level 20 Boss Finale: Apex Predator + Elite Herd
        targetSpeciesKey = speciesList[3];
        targetCount = 2;
        ambientCount = 3;
    }

    const baseDistance = 65 + levelNum * 5.5 + worldIdx * 12;
    const windMax = 1.5 + levelNum * 0.25 + worldIdx * 0.8;

    return {
        worldIdx,
        levelNum,
        targetSpeciesKey,
        targetCount,
        ambientCount,
        baseDistance,
        windMax,
        isBoss: levelNum === 20
    };
}

// ----------------------------------------------------------------------------
// 2. SAVE STATE & PROGRESSION MANAGER
// ----------------------------------------------------------------------------
const SAVE_KEY = "apex_horizon_3d_save_v1";

let saveData = {
    cash: 500,
    xp: 0,
    unlockedWorlds: [true, false, false, false],
    maxLevelByWorld: [1, 1, 1, 1], // 1..20 (21 means world beaten)
    equippedGun: "rifle_243",
    ownedGuns: {
        rifle_243: { power: 0, zoom: 0, stability: 0, reload: 0 }
    }
};

function loadSave() {
    try {
        const raw = localStorage.getItem(SAVE_KEY);
        if (raw) {
            const parsed = JSON.parse(raw);
            saveData = { ...saveData, ...parsed };
        }
    } catch (e) {
        console.warn("Save load fallback:", e);
    }
}

function writeSave() {
    try {
        localStorage.setItem(SAVE_KEY, JSON.stringify(saveData));
    } catch (e) {
        console.warn("Save write error:", e);
    }
}

function getGunStats(gunId) {
    const base = GUNS_DB.find(g => g.id === gunId) || GUNS_DB[0];
    const upg = saveData.ownedGuns[gunId] || { power: 0, zoom: 0, stability: 0, reload: 0 };
    return {
        ...base,
        upgrades: upg,
        power: Math.round(base.basePower * (1 + upg.power * 0.18)),
        zoomMin: +(base.baseZoomMin + upg.zoom * 0.5).toFixed(1),
        zoomMax: +(base.baseZoomMax + upg.zoom * 1.5).toFixed(1),
        stability: Math.min(0.97, base.baseStability + upg.stability * 0.08),
        reloadTime: +(base.baseReload * Math.pow(0.88, upg.reload)).toFixed(2)
    };
}

// ----------------------------------------------------------------------------
// 3. PROCEDURAL WEB AUDIO ENGINE (Gunshot, Echo, Wind, Heartbeat, Impact)
// ----------------------------------------------------------------------------
class SoundEngine {
    constructor() {
        this.ctx = null;
        this.windGain = null;
        this.heartbeatTimer = 0;
    }

    init() {
        if (!this.ctx) {
            const AudioCtx = window.AudioContext || window.webkitAudioContext;
            this.ctx = new AudioCtx();
            this.startWindAmbience();
        }
        if (this.ctx.state === "suspended") {
            this.ctx.resume();
        }
    }

    startWindAmbience() {
        if (!this.ctx) return;
        const bufferSize = this.ctx.sampleRate * 3;
        const buffer = this.ctx.createBuffer(1, bufferSize, this.ctx.sampleRate);
        const data = buffer.getChannelData(0);
        for (let i = 0; i < bufferSize; i++) {
            data[i] = Math.random() * 2 - 1;
        }
        const noise = this.ctx.createBufferSource();
        noise.buffer = buffer;
        noise.loop = true;

        const filter = this.ctx.createBiquadFilter();
        filter.type = "bandpass";
        filter.frequency.value = 320;
        filter.Q.value = 1.4;

        this.windGain = this.ctx.createGain();
        this.windGain.gain.value = 0.035;

        noise.connect(filter);
        filter.connect(this.windGain);
        this.windGain.connect(this.ctx.destination);
        noise.start();
    }

    setWindVolume(speed) {
        if (this.windGain && this.ctx) {
            const target = Math.min(0.11, 0.02 + speed * 0.012);
            this.windGain.gain.setTargetAtTime(target, this.ctx.currentTime, 0.3);
        }
    }

    playGunshot(power = 100) {
        this.init();
        const now = this.ctx.currentTime;

        // 1. Sub-bass punch
        const osc = this.ctx.createOscillator();
        const oscGain = this.ctx.createGain();
        osc.type = "triangle";
        osc.frequency.setValueAtTime(140 + power * 0.2, now);
        osc.frequency.exponentialRampToValueAtTime(28, now + 0.28);
        oscGain.gain.setValueAtTime(0.85, now);
        oscGain.gain.exponentialRampToValueAtTime(0.001, now + 0.32);
        osc.connect(oscGain);
        oscGain.connect(this.ctx.destination);
        osc.start(now);
        osc.stop(now + 0.34);

        // 2. High-pressure crack + canyon reverb tail
        const duration = 1.45;
        const bufferSize = this.ctx.sampleRate * duration;
        const noiseBuffer = this.ctx.createBuffer(1, bufferSize, this.ctx.sampleRate);
        const output = noiseBuffer.getChannelData(0);
        for (let i = 0; i < bufferSize; i++) {
            const t = i / this.ctx.sampleRate;
            const env = t < 0.06 ? Math.exp(-t * 28) : 0.28 * Math.exp(-(t - 0.06) * 3.1);
            output[i] = (Math.random() * 2 - 1) * env;
        }

        const whiteNoise = this.ctx.createBufferSource();
        whiteNoise.buffer = noiseBuffer;

        const filter = this.ctx.createBiquadFilter();
        filter.type = "lowpass";
        filter.frequency.setValueAtTime(3400, now);
        filter.frequency.exponentialRampToValueAtTime(320, now + duration);

        const gain = this.ctx.createGain();
        gain.gain.setValueAtTime(0.95, now);
        gain.gain.exponentialRampToValueAtTime(0.001, now + duration);

        whiteNoise.connect(filter);
        filter.connect(gain);
        gain.connect(this.ctx.destination);
        whiteNoise.start(now);
    }

    playDryFire() {
        this.init();
        const now = this.ctx.currentTime;
        const osc = this.ctx.createOscillator();
        const gain = this.ctx.createGain();
        osc.type = "square";
        osc.frequency.setValueAtTime(950, now);
        osc.frequency.exponentialRampToValueAtTime(240, now + 0.04);
        gain.gain.setValueAtTime(0.25, now);
        gain.gain.exponentialRampToValueAtTime(0.001, now + 0.045);
        osc.connect(gain);
        gain.connect(this.ctx.destination);
        osc.start(now);
        osc.stop(now + 0.05);
    }

    playReload() {
        this.init();
        const now = this.ctx.currentTime;
        [0, 0.25].forEach((offset, idx) => {
            const osc = this.ctx.createOscillator();
            const gain = this.ctx.createGain();
            osc.type = "sawtooth";
            osc.frequency.setValueAtTime(idx === 0 ? 480 : 680, now + offset);
            osc.frequency.exponentialRampToValueAtTime(180, now + offset + 0.09);
            gain.gain.setValueAtTime(0.22, now + offset);
            gain.gain.exponentialRampToValueAtTime(0.001, now + offset + 0.1);
            osc.connect(gain);
            gain.connect(this.ctx.destination);
            osc.start(now + offset);
            osc.stop(now + offset + 0.11);
        });
    }

    playHitMarker(isVital = false) {
        this.init();
        const now = this.ctx.currentTime;
        const osc = this.ctx.createOscillator();
        const gain = this.ctx.createGain();
        osc.type = isVital ? "sine" : "triangle";
        osc.frequency.setValueAtTime(isVital ? 980 : 520, now);
        osc.frequency.exponentialRampToValueAtTime(isVital ? 1420 : 310, now + 0.14);
        gain.gain.setValueAtTime(0.4, now);
        gain.gain.exponentialRampToValueAtTime(0.001, now + 0.16);
        osc.connect(gain);
        gain.connect(this.ctx.destination);
        osc.start(now);
        osc.stop(now + 0.17);
    }

    playHeartbeat() {
        this.init();
        const now = this.ctx.currentTime;
        [0, 0.16].forEach((delay, idx) => {
            const osc = this.ctx.createOscillator();
            const gain = this.ctx.createGain();
            osc.type = "sine";
            osc.frequency.setValueAtTime(idx === 0 ? 68 : 55, now + delay);
            osc.frequency.exponentialRampToValueAtTime(32, now + delay + 0.12);
            gain.gain.setValueAtTime(idx === 0 ? 0.35 : 0.22, now + delay);
            gain.gain.exponentialRampToValueAtTime(0.001, now + delay + 0.13);
            osc.connect(gain);
            gain.connect(this.ctx.destination);
            osc.start(now + delay);
            osc.stop(now + delay + 0.14);
        });
    }
}

const audio = new SoundEngine();

// ----------------------------------------------------------------------------
// 4. THREE.JS 3D SCENE, ENVIRONMENT & ANIMAL BUILDER
// ----------------------------------------------------------------------------
const canvas = document.getElementById("gameCanvas");
const renderer = new THREE.WebGLRenderer({ canvas, antialias: true, powerPreference: "high-performance" });
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.2, 650);

// Lights
const hemiLight = new THREE.HemisphereLight(0xffffff, 0x3b4d34, 0.85);
scene.add(hemiLight);

const dirLight = new THREE.DirectionalLight(0xfff3d6, 1.15);
dirLight.position.set(80, 140, 60);
dirLight.castShadow = true;
dirLight.shadow.mapSize.width = 1024;
dirLight.shadow.mapSize.height = 1024;
dirLight.shadow.camera.near = 10;
dirLight.shadow.camera.far = 380;
const dShadow = 140;
dirLight.shadow.camera.left = -dShadow;
dirLight.shadow.camera.right = dShadow;
dirLight.shadow.camera.top = dShadow;
dirLight.shadow.camera.bottom = -dShadow;
scene.add(dirLight);

// Groups for clean world transitions
const envGroup = new THREE.Group();
const animalGroup = new THREE.Group();
const fxGroup = new THREE.Group();
scene.add(envGroup);
scene.add(animalGroup);
scene.add(fxGroup);

// Procedural terrain height function (gentle rolling hills)
function getTerrainHeight(x, z) {
    const distFromOrigin = Math.sqrt(x * x + z * z);
    const valleyFlatten = Math.min(1, Math.max(0, (distFromOrigin - 12) / 35));
    const h1 = Math.sin(x * 0.025) * Math.cos(z * 0.025) * 4.2;
    const h2 = Math.sin(x * 0.06 + 1.4) * Math.sin(z * 0.05) * 1.6;
    return (h1 + h2) * valleyFlatten;
}

let weatherParticles = null;
let swayingTrees = [];

function buildEnvironment(worldIdx) {
    // Clear previous environment
    while (envGroup.children.length > 0) {
        envGroup.remove(envGroup.children[0]);
    }
    swayingTrees = [];
    weatherParticles = null;

    const w = WORLDS[worldIdx];
    scene.background = new THREE.Color(w.skyColor);
    scene.fog = new THREE.FogExp2(w.fogColor, w.fogDensity);
    dirLight.color.setHex(w.sunColor);

    // Terrain Mesh
    const geo = new THREE.PlaneGeometry(520, 520, 72, 72);
    geo.rotateX(-Math.PI / 2);
    const pos = geo.attributes.position;
    const colors = [];
    const c1 = new THREE.Color(w.groundColor);
    const c2 = new THREE.Color(w.groundAccent);

    for (let i = 0; i < pos.count; i++) {
        const vx = pos.getX(i);
        const vz = pos.getZ(i);
        const vy = getTerrainHeight(vx, vz);
        pos.setY(i, vy);

        const blend = (Math.sin(vx * 0.05) * Math.cos(vz * 0.05) + 1) * 0.5;
        const c = c1.clone().lerp(c2, blend);
        colors.push(c.r, c.g, c.b);
    }
    geo.setAttribute("color", new THREE.Float32BufferAttribute(colors, 3));
    geo.computeVertexNormals();

    const terrainMat = new THREE.MeshStandardMaterial({
        vertexColors: true,
        roughness: 0.92,
        metalness: 0.04
    });
    const terrain = new THREE.Mesh(geo, terrainMat);
    terrain.receiveShadow = true;
    envGroup.add(terrain);

    // Distant Mountain Rim
    const rimGeo = new THREE.CylinderGeometry(235, 255, 48, 32, 1, true);
    const rimMat = new THREE.MeshStandardMaterial({
        color: new THREE.Color(w.fogColor).multiplyScalar(0.75),
        side: THREE.BackSide,
        roughness: 1
    });
    const rim = new THREE.Mesh(rimGeo, rimMat);
    rim.position.y = 18;
    envGroup.add(rim);

    // Populate Flora & Rocks
    const treeCount = w.treeType === "acacia" ? 45 : 85;
    for (let i = 0; i < treeCount; i++) {
        const angle = Math.random() * Math.PI * 2;
        const radius = 26 + Math.random() * 185;
        const tx = Math.cos(angle) * radius;
        const tz = Math.sin(angle) * radius;
        // Keep central shooting lane slightly open
        if (Math.abs(tx) < 12 && tz < -15 && tz > -140) continue;

        const ty = getTerrainHeight(tx, tz);
        const tree = createBiomeTree(w.treeType);
        tree.position.set(tx, ty, tz);
        const scale = 0.8 + Math.random() * 0.7;
        tree.scale.set(scale, scale, scale);
        envGroup.add(tree);
        swayingTrees.push({ mesh: tree, phase: Math.random() * Math.PI * 2 });
    }

    // Boulders / Rocks
    const rockGeo = new THREE.DodecahedronGeometry(1.4, 0);
    const rockMat = new THREE.MeshStandardMaterial({
        color: worldIdx === 2 ? 0x64748b : 0x57534e,
        roughness: 0.9
    });
    for (let i = 0; i < 40; i++) {
        const angle = Math.random() * Math.PI * 2;
        const radius = 18 + Math.random() * 175;
        const rx = Math.cos(angle) * radius;
        const rz = Math.sin(angle) * radius;
        const ry = getTerrainHeight(rx, rz);
        const rock = new THREE.Mesh(rockGeo, rockMat);
        rock.position.set(rx, ry + 0.5, rz);
        rock.scale.set(0.7 + Math.random() * 1.4, 0.5 + Math.random() * 0.9, 0.7 + Math.random() * 1.4);
        rock.rotation.set(Math.random(), Math.random(), Math.random());
        rock.castShadow = true;
        rock.receiveShadow = true;
        envGroup.add(rock);
    }

    // Weather Particles (Snow, Dust Motes, Mist)
    const pCount = 450;
    const pGeo = new THREE.BufferGeometry();
    const pPositions = new Float32Array(pCount * 3);
    for (let i = 0; i < pCount; i++) {
        pPositions[i * 3] = (Math.random() - 0.5) * 160;
        pPositions[i * 3 + 1] = Math.random() * 45;
        pPositions[i * 3 + 2] = (Math.random() - 0.5) * 160;
    }
    pGeo.setAttribute("position", new THREE.BufferAttribute(pPositions, 3));
    const pColor = w.weather === "snow" ? 0xffffff : (w.weather === "dust" ? 0xfde68a : 0xd1fae5);
    const pMat = new THREE.PointsMaterial({
        color: pColor,
        size: w.weather === "snow" ? 0.35 : 0.22,
        transparent: true,
        opacity: 0.65
    });
    weatherParticles = new THREE.Points(pGeo, pMat);
    envGroup.add(weatherParticles);
}

function createBiomeTree(type) {
    const group = new THREE.Group();
    if (type === "pine" || type === "snowpine") {
        const trunkMat = new THREE.MeshStandardMaterial({ color: 0x4a3525, roughness: 0.9 });
        const trunk = new THREE.Mesh(new THREE.CylinderGeometry(0.25, 0.45, 4.5, 6), trunkMat);
        trunk.position.y = 2.25;
        trunk.castShadow = true;
        group.add(trunk);

        const leafColor = type === "snowpine" ? 0xdbeafe : 0x1e3f20;
        const leafMat = new THREE.MeshStandardMaterial({ color: leafColor, roughness: 0.85 });
        for (let i = 0; i < 3; i++) {
            const cone = new THREE.Mesh(new THREE.ConeGeometry(2.4 - i * 0.55, 4.2 - i * 0.4, 7), leafMat);
            cone.position.y = 4.6 + i * 2.1;
            cone.castShadow = true;
            group.add(cone);
        }
    } else if (type === "acacia") {
        const trunkMat = new THREE.MeshStandardMaterial({ color: 0x5c4632, roughness: 0.9 });
        const trunk = new THREE.Mesh(new THREE.CylinderGeometry(0.22, 0.4, 5.2, 6), trunkMat);
        trunk.position.y = 2.6;
        trunk.castShadow = true;
        group.add(trunk);

        const canopyMat = new THREE.MeshStandardMaterial({ color: 0x5d6e32, roughness: 0.85 });
        const canopy = new THREE.Mesh(new THREE.CylinderGeometry(4.4, 2.2, 1.1, 9), canopyMat);
        canopy.position.y = 5.4;
        canopy.castShadow = true;
        group.add(canopy);
    } else if (type === "bamboo") {
        const stalkMat = new THREE.MeshStandardMaterial({ color: 0x4d7c3f, roughness: 0.6 });
        for (let s = 0; s < 4; s++) {
            const ox = (Math.random() - 0.5) * 1.1;
            const oz = (Math.random() - 0.5) * 1.1;
            const stalk = new THREE.Mesh(new THREE.CylinderGeometry(0.11, 0.14, 9.5, 6), stalkMat);
            stalk.position.set(ox, 4.75, oz);
            stalk.castShadow = true;
            group.add(stalk);
        }
        const leafMat = new THREE.MeshStandardMaterial({ color: 0x65a30d, roughness: 0.8 });
        const crown = new THREE.Mesh(new THREE.SphereGeometry(1.8, 7, 6), leafMat);
        crown.scale.set(1.3, 0.6, 1.3);
        crown.position.y = 8.8;
        group.add(crown);
    }
    return group;
}

// ----------------------------------------------------------------------------
// 5. ARTICULATED 3D ANIMAL ENTITY & ANATOMICAL HITBOXES
// ----------------------------------------------------------------------------
class AnimalEntity {
    constructor(speciesKey, isTarget, position) {
        this.speciesKey = speciesKey;
        this.spec = SPECIES_DB[speciesKey];
        this.isTarget = isTarget;
        this.maxHp = this.spec.hp;
        this.hp = this.maxHp;
        this.alive = true;

        // AI State: 'graze', 'patrol', 'alert', 'flee', 'charge'
        this.state = Math.random() < 0.5 ? "graze" : "patrol";
        this.stateTimer = 2 + Math.random() * 4;
        this.heading = Math.random() * Math.PI * 2;
        this.animPhase = Math.random() * 10;

        this.root = new THREE.Group();
        this.root.position.copy(position);
        this.hitboxes = []; // { mesh, zone: 'head' | 'vital' | 'body' }
        this.legs = [];

        this.buildMesh();
        animalGroup.add(this.root);
    }

    buildMesh() {
        const s = this.spec.scale;
        const bodyMat = new THREE.MeshStandardMaterial({ color: this.spec.bodyColor, roughness: 0.82 });
        const accentMat = new THREE.MeshStandardMaterial({ color: this.spec.accentColor, roughness: 0.85 });

        // Torso
        const torsoGeo = new THREE.BoxGeometry(1.05 * s, 1.05 * s, 2.15 * s);
        const torso = new THREE.Mesh(torsoGeo, bodyMat);
        torso.position.set(0, 1.45 * s, 0);
        torso.castShadow = true;
        torso.receiveShadow = true;
        this.root.add(torso);
        this.hitboxes.push({ mesh: torso, zone: "body", animal: this });

        // Vital Organ Zone (Heart & Lungs in forward chest)
        const vitalGeo = new THREE.SphereGeometry(0.54 * s, 10, 10);
        const vitalMat = new THREE.MeshBasicMaterial({ visible: false });
        const vitalMesh = new THREE.Mesh(vitalGeo, vitalMat);
        vitalMesh.position.set(0, 1.42 * s, 0.55 * s);
        this.root.add(vitalMesh);
        this.hitboxes.push({ mesh: vitalMesh, zone: "vital", animal: this });

        // Neck & Head Group
        this.headPivot = new THREE.Group();
        this.headPivot.position.set(0, 1.85 * s, 0.95 * s);
        this.root.add(this.headPivot);

        const neck = new THREE.Mesh(new THREE.BoxGeometry(0.55 * s, 0.75 * s, 0.65 * s), bodyMat);
        neck.position.set(0, 0.25 * s, 0.2 * s);
        neck.rotation.x = 0.35;
        neck.castShadow = true;
        this.headPivot.add(neck);

        const headMesh = new THREE.Mesh(new THREE.BoxGeometry(0.58 * s, 0.56 * s, 0.82 * s), bodyMat);
        headMesh.position.set(0, 0.58 * s, 0.55 * s);
        headMesh.castShadow = true;
        this.headPivot.add(headMesh);
        this.hitboxes.push({ mesh: headMesh, zone: "head", animal: this });

        // Snout / Muzzle
        const snout = new THREE.Mesh(new THREE.BoxGeometry(0.36 * s, 0.32 * s, 0.42 * s), accentMat);
        snout.position.set(0, 0.48 * s, 0.98 * s);
        this.headPivot.add(snout);

        // Antlers / Horns / Mane
        if (this.spec.antlers === "deer" || this.spec.antlers === "elk") {
            const antMat = new THREE.MeshStandardMaterial({ color: 0xe5dec9, roughness: 0.7 });
            const h = this.spec.antlers === "elk" ? 1.15 * s : 0.75 * s;
            [-1, 1].forEach(side => {
                const mainBeam = new THREE.Mesh(new THREE.CylinderGeometry(0.04 * s, 0.06 * s, h, 5), antMat);
                mainBeam.position.set(side * 0.26 * s, 0.95 * s, 0.45 * s);
                mainBeam.rotation.z = -side * 0.35;
                mainBeam.rotation.x = -0.2;
                this.headPivot.add(mainBeam);

                const tine = new THREE.Mesh(new THREE.CylinderGeometry(0.03 * s, 0.04 * s, h * 0.55, 5), antMat);
                tine.position.set(side * 0.34 * s, 1.05 * s, 0.65 * s);
                tine.rotation.x = 0.45;
                this.headPivot.add(tine);
            });
        } else if (this.spec.antlers === "horns" || this.spec.antlers === "buffalo") {
            const hornMat = new THREE.MeshStandardMaterial({ color: 0x292524, roughness: 0.6 });
            [-1, 1].forEach(side => {
                const horn = new THREE.Mesh(new THREE.ConeGeometry(0.09 * s, 0.75 * s, 6), hornMat);
                horn.position.set(side * 0.34 * s, 0.85 * s, 0.48 * s);
                horn.rotation.z = -side * (this.spec.antlers === "buffalo" ? 1.1 : 0.25);
                this.headPivot.add(horn);
            });
        } else if (this.spec.antlers === "mane") {
            const mane = new THREE.Mesh(new THREE.BoxGeometry(0.88 * s, 0.92 * s, 0.85 * s), accentMat);
            mane.position.set(0, 0.45 * s, 0.28 * s);
            this.headPivot.add(mane);
        }

        // Four Articulated Legs
        const legGeo = new THREE.BoxGeometry(0.26 * s, 1.15 * s, 0.28 * s);
        const legOffsets = [
            [-0.35 * s, 0.58 * s, 0.72 * s],
            [0.35 * s, 0.58 * s, 0.72 * s],
            [-0.35 * s, 0.58 * s, -0.72 * s],
            [0.35 * s, 0.58 * s, -0.72 * s]
        ];
        legOffsets.forEach((off, idx) => {
            const leg = new THREE.Mesh(legGeo, bodyMat);
            leg.position.set(off[0], off[1], off[2]);
            leg.castShadow = true;
            this.root.add(leg);
            this.legs.push({ mesh: leg, idx });
        });

        // Subtle Target Indicator Ring on Ground (only for primary contract targets)
        if (this.isTarget) {
            const ringGeo = new THREE.RingGeometry(1.1 * s, 1.3 * s, 24);
            ringGeo.rotateX(-Math.PI / 2);
            const ringMat = new THREE.MeshBasicMaterial({
                color: 0xf59e0b,
                transparent: true,
                opacity: 0.32,
                side: THREE.DoubleSide
            });
            this.targetRing = new THREE.Mesh(ringGeo, ringMat);
            this.targetRing.position.y = 0.08;
            this.root.add(this.targetRing);
        }
    }

    update(dt, playerPos, windVec) {
        if (!this.alive) {
            // Fall over smoothly on death
            if (this.root.rotation.z < Math.PI * 0.46) {
                this.root.rotation.z += dt * 3.2;
                this.root.position.y = Math.max(
                    getTerrainHeight(this.root.position.x, this.root.position.z) + 0.35,
                    this.root.position.y - dt * 1.4
                );
            }
            return;
        }

        this.stateTimer -= dt;
        const distToPlayer = this.root.position.distanceTo(playerPos);

        // Check if wind carries player scent toward animal
        const toAnimal = new THREE.Vector3().subVectors(this.root.position, playerPos).normalize();
        const windAlignment = toAnimal.dot(windVec.clone().normalize());
        if (this.state === "graze" && windAlignment > 0.85 && distToPlayer < 85 && Math.random() < dt * 0.25) {
            this.state = "alert";
            this.stateTimer = 2.5;
        }

        // State transitions
        if (this.stateTimer <= 0) {
            if (this.state === "graze") {
                this.state = "patrol";
                this.stateTimer = 3 + Math.random() * 4;
                this.heading += (Math.random() - 0.5) * 1.8;
            } else if (this.state === "patrol") {
                this.state = "graze";
                this.stateTimer = 2.5 + Math.random() * 3.5;
            } else if (this.state === "alert") {
                this.state = "flee";
                this.stateTimer = 6;
                this.heading = Math.atan2(this.root.position.x - playerPos.x, this.root.position.z - playerPos.z);
            } else if (this.state === "flee") {
                this.state = "patrol";
                this.stateTimer = 4;
            }
        }

        // Movement & Animation
        let moveSpeed = 0;
        if (this.state === "patrol") {
            moveSpeed = this.spec.speed * 0.26;
        } else if (this.state === "flee") {
            moveSpeed = this.spec.speed * 0.95;
            // Steer away from player
            const awayAngle = Math.atan2(this.root.position.x - playerPos.x, this.root.position.z - playerPos.z);
            this.heading = awayAngle;
        } else if (this.state === "charge") {
            moveSpeed = this.spec.speed * 1.05;
            const towardAngle = Math.atan2(playerPos.x - this.root.position.x, playerPos.z - this.root.position.z);
            this.heading = towardAngle;
        }

        if (moveSpeed > 0) {
            this.root.position.x += Math.sin(this.heading) * moveSpeed * dt;
            this.root.position.z += Math.cos(this.heading) * moveSpeed * dt;

            // Keep inside hunting preserve bounds
            const distOrigin = Math.hypot(this.root.position.x, this.root.position.z);
            if (distOrigin > 195) {
                this.heading = Math.atan2(-this.root.position.x, -this.root.position.z);
            }
            if (distOrigin < 14 && this.state !== "charge") {
                this.heading = Math.atan2(this.root.position.x, this.root.position.z);
            }

            this.animPhase += dt * moveSpeed * 1.35;
            this.legs.forEach(l => {
                const dir = (l.idx === 0 || l.idx === 3) ? 1 : -1;
                l.mesh.rotation.x = Math.sin(this.animPhase) * 0.55 * dir;
            });
            this.headPivot.rotation.x = 0;
        } else {
            // Reset legs & animate grazing/alert head
            this.legs.forEach(l => l.mesh.rotation.x *= 0.85);
            if (this.state === "graze") {
                this.headPivot.rotation.x = 0.55 + Math.sin(performance.now() * 0.003) * 0.08;
            } else {
                this.headPivot.rotation.x = -0.1;
            }
        }

        // Smooth rotation & ground snap
        this.root.rotation.y = this.heading;
        const groundY = getTerrainHeight(this.root.position.x, this.root.position.z);
        this.root.position.y = groundY;

        if (this.targetRing) {
            this.targetRing.rotation.z += dt * 0.8;
        }
    }

    alertNearby(soundPos, isDirectHit = false) {
        if (!this.alive) return;
        const dist = this.root.position.distanceTo(soundPos);
        if (isDirectHit) {
            if (this.spec.predator && dist < 130) {
                this.state = "charge";
                this.stateTimer = 12;
            } else {
                this.state = "flee";
                this.stateTimer = 7;
            }
        } else if (dist < 145) {
            if (this.state === "graze" || this.state === "patrol") {
                this.state = "flee";
                this.stateTimer = 5.5;
            }
        }
    }
}

// ----------------------------------------------------------------------------
// 6. GAMEPLAY STATE, AIMING, BALLISTICS & HUD CONTROLLERS
// ----------------------------------------------------------------------------
const gameState = {
    active: false,
    paused: false,
    worldIdx: 0,
    levelNum: 1,
    levelConfig: null,
    animals: [],
    targetsRemaining: 0,
    roundEarnedBase: 0,
    roundEarnedVital: 0,
    roundEarnedDist: 0,

    // Player Camera & Scope
    yaw: 0,
    pitch: 0.02,
    isScoped: false,
    currentZoom: 4.0,
    holdingBreath: false,
    breathStamina: 100, // 0..100
    breathExhausted: false,
    swayTime: 0,
    recoilPitch: 0,

    // Weapon Ammo & Reload
    ammo: 4,
    isReloading: false,
    reloadTimer: 0,

    // Wind
    windSpeed: 3.5,
    windAngle: 0, // radians
    windVec: new THREE.Vector3(1, 0, 0),

    // Active tracers
    tracers: []
};

const playerOrigin = new THREE.Vector3(0, 2.2, 0);
camera.position.copy(playerOrigin);

// Raycaster for rangefinder & bullet impact
const raycaster = new THREE.Raycaster();

function startExpedition(worldIdx, levelNum) {
    audio.init();
    gameState.worldIdx = worldIdx;
    gameState.levelNum = levelNum;
    gameState.levelConfig = getLevelConfig(worldIdx, levelNum);
    gameState.active = true;
    gameState.paused = false;

    // Reset Round Stats
    gameState.roundEarnedBase = 0;
    gameState.roundEarnedVital = 0;
    gameState.roundEarnedDist = 0;

    // Reset Camera & Scope
    gameState.yaw = 0;
    gameState.pitch = 0.01;
    setScoped(false);
    const gun = getGunStats(saveData.equippedGun);
    gameState.currentZoom = gun.zoomMin;
    gameState.ammo = gun.magSize;
    gameState.isReloading = false;
    gameState.breathStamina = 100;
    gameState.breathExhausted = false;

    // Generate Crosswind
    gameState.windSpeed = +(0.8 + Math.random() * gameState.levelConfig.windMax).toFixed(1);
    gameState.windAngle = Math.random() * Math.PI * 2;
    gameState.windVec.set(Math.cos(gameState.windAngle), 0, Math.sin(gameState.windAngle));
    audio.setWindVolume(gameState.windSpeed);

    // Build 3D World
    buildEnvironment(worldIdx);

    // Spawn Wildlife
    while (animalGroup.children.length > 0) {
        animalGroup.remove(animalGroup.children[0]);
    }
    gameState.animals = [];

    const cfg = gameState.levelConfig;
    gameState.targetsRemaining = cfg.targetCount;

    // Spawn Primary Contract Targets in front hemisphere (-Z)
    for (let i = 0; i < cfg.targetCount; i++) {
        const spreadAngle = ((i - (cfg.targetCount - 1) / 2) * 0.38) + (Math.random() - 0.5) * 0.25;
        const dist = cfg.baseDistance + (Math.random() - 0.5) * 24;
        const ax = Math.sin(spreadAngle) * dist;
        const az = -Math.cos(spreadAngle) * dist;
        const ay = getTerrainHeight(ax, az);
        const entity = new AnimalEntity(cfg.targetSpeciesKey, true, new THREE.Vector3(ax, ay, az));
        gameState.animals.push(entity);
    }

    // Spawn Ambient Wildlife
    const worldSpecies = WORLDS[worldIdx].animals;
    for (let i = 0; i < cfg.ambientCount; i++) {
        const randSpec = worldSpecies[Math.floor(Math.random() * worldSpecies.length)];
        const angle = (Math.random() - 0.5) * Math.PI * 1.4;
        const dist = 55 + Math.random() * 110;
        const ax = Math.sin(angle) * dist;
        const az = -Math.cos(angle) * dist;
        const ay = getTerrainHeight(ax, az);
        const entity = new AnimalEntity(randSpec, false, new THREE.Vector3(ax, ay, az));
        gameState.animals.push(entity);
    }

    // Toggle UI
    document.getElementById("mainMenuOverlay").classList.add("hidden");
    document.getElementById("roundEndModal").classList.add("hidden");
    document.getElementById("roundEndModal").classList.remove("flex");
    document.getElementById("gameHud").classList.remove("hidden");
    document.getElementById("dangerOverlay").classList.add("hidden");

    updateHud();
    showBanner(`Contract: Harvest ${cfg.targetCount}x ${SPECIES_DB[cfg.targetSpeciesKey].name}`);
}

function setScoped(scoped) {
    gameState.isScoped = scoped;
    const scopeOverlay = document.getElementById("scopeOverlay");
    const hipCrosshair = document.getElementById("hipCrosshair");
    const gun = getGunStats(saveData.equippedGun);

    if (scoped) {
        scopeOverlay.style.display = "block";
        hipCrosshair.style.display = "none";
        camera.fov = 60 / gameState.currentZoom;
    } else {
        scopeOverlay.style.display = "none";
        hipCrosshair.style.display = "flex";
        camera.fov = 60;
        gameState.holdingBreath = false;
    }
    camera.updateProjectionMatrix();
    document.getElementById("scopeZoomVal").textContent = gameState.currentZoom.toFixed(1);
}

function cycleZoom(delta = 1) {
    const gun = getGunStats(saveData.equippedGun);
    if (!gameState.isScoped) {
        setScoped(true);
        return;
    }
    const step = (gun.zoomMax - gun.zoomMin) / 3;
    gameState.currentZoom += delta * step;
    if (gameState.currentZoom > gun.zoomMax + 0.01) gameState.currentZoom = gun.zoomMin;
    if (gameState.currentZoom < gun.zoomMin - 0.01) gameState.currentZoom = gun.zoomMax;
    camera.fov = 60 / gameState.currentZoom;
    camera.updateProjectionMatrix();
    document.getElementById("scopeZoomVal").textContent = gameState.currentZoom.toFixed(1);
}

function reloadWeapon() {
    const gun = getGunStats(saveData.equippedGun);
    if (gameState.isReloading || gameState.ammo >= gun.magSize) return;
    gameState.isReloading = true;
    gameState.reloadTimer = gun.reloadTime;
    setScoped(false);
    audio.playReload();
    updateHud();
}

function fireShot() {
    if (!gameState.active || gameState.paused) return;
    const gun = getGunStats(saveData.equippedGun);

    if (gameState.isReloading) return;
    if (gameState.ammo <= 0) {
        audio.playDryFire();
        reloadWeapon();
        return;
    }

    gameState.ammo--;
    audio.playGunshot(gun.power);

    // Apply physical recoil
    gameState.recoilPitch += (0.018 + (gun.power / 12000)) * (1.25 - gun.stability * 0.5);
    gameState.yaw += (Math.random() - 0.5) * 0.008;

    // Calculate Ballistic Trajectory from Center of Camera
    const forward = new THREE.Vector3(0, 0, -1).applyQuaternion(camera.quaternion).normalize();
    const right = new THREE.Vector3(1, 0, 0).applyQuaternion(camera.quaternion).normalize();
    const up = new THREE.Vector3(0, 1, 0).applyQuaternion(camera.quaternion).normalize();

    // Estimate target range along aim line for ballistic drop & crosswind offset
    raycaster.set(camera.position, forward);
    const allHitMeshes = [];
    gameState.animals.forEach(a => {
        if (a.alive) a.hitboxes.forEach(hb => allHitMeshes.push(hb.mesh));
    });

    // Check approximate distance to aim point
    let estDist = 110;
    const roughHits = raycaster.intersectObjects(allHitMeshes, false);
    if (roughHits.length > 0) {
        estDist = roughHits[0].distance;
    }

    // Time of flight & environmental deflection
    const flightTime = estDist / gun.muzzleVel;
    const gravityDrop = 0.5 * 9.81 * flightTime * flightTime * 0.52;
    const crosswindDot = gameState.windVec.dot(right);
    const windDrift = crosswindDot * gameState.windSpeed * flightTime * 0.38;

    // Slight hip-fire dispersion if not scoped
    const hipSpread = gameState.isScoped ? 0.0004 : 0.014;
    const spreadX = (Math.random() - 0.5) * hipSpread;
    const spreadY = (Math.random() - 0.5) * hipSpread;

    // Final ballistic ray direction
    const impactWorldGuess = camera.position.clone()
        .addScaledVector(forward, estDist)
        .addScaledVector(up, -gravityDrop + spreadY * estDist)
        .addScaledVector(right, windDrift + spreadX * estDist);

    const ballisticDir = impactWorldGuess.sub(camera.position).normalize();
    raycaster.set(camera.position, ballisticDir);

    const hits = raycaster.intersectObjects(allHitMeshes, false);
    let tracerEnd = camera.position.clone().addScaledVector(ballisticDir, 220);

    if (hits.length > 0) {
        // Prioritize vital/head hitboxes if ray passes through both body and vital sphere
        let chosenHit = hits[0];
        for (const h of hits) {
            const hbInfo = findHitboxInfo(h.object);
            if (hbInfo && (hbInfo.zone === "head" || hbInfo.zone === "vital")) {
                chosenHit = h;
                break;
            }
        }

        tracerEnd = chosenHit.point.clone();
        const hbInfo = findHitboxInfo(chosenHit.object);
        if (hbInfo && hbInfo.animal.alive) {
            processAnimalHit(hbInfo.animal, hbInfo.zone, chosenHit.distance, gun);
        }
    } else {
        showFloatingFeedback("MISS — CHECK WIND & DROP", "#94a3b8");
    }

    // Spawn luminous bullet tracer
    spawnBulletTracer(camera.position.clone().addScaledVector(right, 0.18).addScaledVector(up, -0.15), tracerEnd);

    // Alert all wildlife within acoustic radius
    gameState.animals.forEach(a => a.alertNearby(camera.position, false));

    updateHud();
    if (gameState.ammo === 0) {
        setTimeout(() => {
            if (gameState.active && gameState.ammo === 0) reloadWeapon();
        }, 450);
    }
}

function findHitboxInfo(mesh) {
    for (const a of gameState.animals) {
        for (const hb of a.hitboxes) {
            if (hb.mesh === mesh) return hb;
        }
    }
    return null;
}

function processAnimalHit(animal, zone, distance, gun) {
    let mult = 1.0;
    let label = "BODY SHOT";
    let color = "#f8fafc";
    let isVital = false;

    if (zone === "head") {
        mult = 2.6;
        label = `🧠 BRAIN SHOT! (${Math.round(distance)}m)`;
        color = "#fbbf24";
        isVital = true;
    } else if (zone === "vital") {
        mult = 2.1;
        label = `🫀 HEART & LUNGS VITAL! (${Math.round(distance)}m)`;
        color = "#34d399";
        isVital = true;
    } else {
        label = `FLESH HIT (${Math.round(distance)}m)`;
        color = "#f87171";
    }

    const damage = Math.round(gun.power * mult);
    animal.hp -= damage;
    audio.playHitMarker(isVital);

    if (animal.hp <= 0) {
        animal.hp = 0;
        animal.alive = false;
        if (animal.targetRing) animal.targetRing.visible = false;

        const baseCash = animal.spec.reward;
        const vitalBonus = isVital ? (zone === "head" ? 75 : 60) : 0;
        const distBonus = distance > 90 ? Math.round((distance - 90) * 0.6) : 15;

        gameState.roundEarnedBase += baseCash;
        gameState.roundEarnedVital += vitalBonus;
        gameState.roundEarnedDist += distBonus;

        saveData.cash += baseCash + vitalBonus + distBonus;
        saveData.xp += animal.spec.xp + (isVital ? 25 : 0);
        writeSave();

        if (animal.isTarget) {
            gameState.targetsRemaining--;
            showFloatingFeedback(`${label} • HARVESTED +$${baseCash + vitalBonus + distBonus}`, color);
            updateHud();

            if (gameState.targetsRemaining <= 0) {
                setTimeout(() => completeLevel(), 1300);
            }
        } else {
            showFloatingFeedback(`${label} • BONUS GAME +$${baseCash}`, "#38bdf8");
            updateHud();
        }
    } else {
        animal.alertNearby(camera.position, true);
        const actionNote = animal.spec.predator ? "PREDATOR CHARGING!" : "TARGET FLEEING!";
        showFloatingFeedback(`${label} (-${damage} HP) • ${actionNote}`, "#f87171");
    }
}

function spawnBulletTracer(start, end) {
    const geo = new THREE.BufferGeometry().setFromPoints([start, end]);
    const mat = new THREE.LineBasicMaterial({ color: 0xfef08a, transparent: true, opacity: 0.9 });
    const line = new THREE.Line(geo, mat);
    fxGroup.add(line);
    gameState.tracers.push({ line, life: 0.28 });
}

function completeLevel() {
    if (!gameState.active) return;
    gameState.active = false;
    setScoped(false);

    const wIdx = gameState.worldIdx;
    const lvl = gameState.levelNum;

    // Unlock next level or next world
    if (lvl >= saveData.maxLevelByWorld[wIdx]) {
        saveData.maxLevelByWorld[wIdx] = Math.min(21, lvl + 1);
    }
    let worldUnlockedBanner = "";
    if (lvl === 20 && wIdx + 1 < WORLDS.length && !saveData.unlockedWorlds[wIdx + 1]) {
        saveData.unlockedWorlds[wIdx + 1] = true;
        worldUnlockedBanner = ` 🎉 NEW WORLD UNLOCKED: ${WORLDS[wIdx + 1].name.toUpperCase()}!`;
    }
    writeSave();

    // Populate Victory Modal
    const modal = document.getElementById("roundEndModal");
    document.getElementById("roundEndBadge").className = "inline-block px-3 py-1 rounded-full text-xs font-tactical font-bold uppercase tracking-widest bg-emerald-500/20 text-emerald-400 border border-emerald-500/40";
    document.getElementById("roundEndBadge").textContent = lvl === 20 ? "WORLD CHAMPIONSHIP CLEARED" : "EXPEDITION SUCCESSFUL";
    document.getElementById("roundEndTitle").textContent = `LEVEL ${lvl} COMPLETE`;
    document.getElementById("roundEndSubtitle").textContent = `${WORLDS[wIdx].name}${worldUnlockedBanner}`;

    document.getElementById("roundRewardsBox").classList.remove("hidden");
    document.getElementById("rewBase").textContent = gameState.roundEarnedBase;
    document.getElementById("rewVital").textContent = gameState.roundEarnedVital;
    document.getElementById("rewDist").textContent = gameState.roundEarnedDist;
    document.getElementById("rewTotal").textContent = gameState.roundEarnedBase + gameState.roundEarnedVital + gameState.roundEarnedDist;

    const nextBtn = document.getElementById("btnEndNextAction");
    if (lvl < 20) {
        nextBtn.textContent = `LEVEL ${lvl + 1} ▶`;
        nextBtn.onclick = () => startExpedition(wIdx, lvl + 1);
    } else if (wIdx + 1 < WORLDS.length) {
        nextBtn.textContent = `ENTER WORLD ${wIdx + 2} ▶`;
        nextBtn.onclick = () => startExpedition(wIdx + 1, 1);
    } else {
        nextBtn.textContent = `REPLAY LEVEL 20 ▶`;
        nextBtn.onclick = () => startExpedition(wIdx, 20);
    }

    modal.classList.remove("hidden");
    modal.classList.add("flex");
}

function failLevel(reasonText) {
    if (!gameState.active) return;
    gameState.active = false;
    setScoped(false);
    document.getElementById("dangerOverlay").classList.add("hidden");

    const modal = document.getElementById("roundEndModal");
    document.getElementById("roundEndBadge").className = "inline-block px-3 py-1 rounded-full text-xs font-tactical font-bold uppercase tracking-widest bg-red-500/20 text-red-400 border border-red-500/40";
    document.getElementById("roundEndBadge").textContent = "EXPEDITION FAILED";
    document.getElementById("roundEndTitle").textContent = "HUNTER OVERRUN";
    document.getElementById("roundEndSubtitle").textContent = reasonText;
    document.getElementById("roundRewardsBox").classList.add("hidden");

    const nextBtn = document.getElementById("btnEndNextAction");
    nextBtn.textContent = "RETRY HUNT ↻";
    nextBtn.onclick = () => startExpedition(gameState.worldIdx, gameState.levelNum);

    modal.classList.remove("hidden");
    modal.classList.add("flex");
}

// ----------------------------------------------------------------------------
// 7. HUD, FEEDBACK & BASE CAMP MENU RENDERING
// ----------------------------------------------------------------------------
function updateHud() {
    const w = WORLDS[gameState.worldIdx];
    const cfg = gameState.levelConfig;
    const gun = getGunStats(saveData.equippedGun);

    document.getElementById("hudWorldBadge").textContent = `WORLD ${w.id + 1}`;
    document.getElementById("hudWorldName").textContent = w.name;
    document.getElementById("hudLevelNum").textContent = gameState.levelNum;
    document.getElementById("hudTargetText").textContent = `${gameState.targetsRemaining}x ${SPECIES_DB[cfg.targetSpeciesKey].name}`;

    document.getElementById("hudWindSpeed").textContent = gameState.windSpeed.toFixed(1);
    const deg = Math.round((gameState.windAngle * 180) / Math.PI);
    document.getElementById("windArrowSvg").style.transform = `rotate(${deg + 90}deg)`;
    const dirs = ["E", "SE", "S", "SW", "W", "NW", "N", "NE"];
    const dirIdx = Math.round(((deg % 360 + 360) % 360) / 45) % 8;
    document.getElementById("hudWindDirText").textContent = dirs[dirIdx];

    document.getElementById("hudCash").textContent = saveData.cash.toLocaleString();
    document.getElementById("hudXp").textContent = saveData.xp.toLocaleString();

    document.getElementById("hudGunName").textContent = gun.name;
    document.getElementById("hudGunSpecs").textContent = `Power: ${gun.power} | Zoom: ${gun.zoomMin}x-${gun.zoomMax}x`;
    document.getElementById("hudAmmoDisplay").innerHTML = `${gameState.ammo} <span class="text-xs text-slate-400">/ ${gun.magSize}</span>`;

    const reloadEl = document.getElementById("hudReloadStatus");
    if (gameState.isReloading) {
        reloadEl.textContent = "RELOADING...";
        reloadEl.className = "text-[10px] font-tactical uppercase tracking-widest text-amber-400 mt-0.5";
    } else if (gameState.ammo === 0) {
        reloadEl.textContent = "EMPTY [R]";
        reloadEl.className = "text-[10px] font-tactical uppercase tracking-widest text-red-400 mt-0.5";
    } else {
        reloadEl.textContent = "CHAMBERED";
        reloadEl.className = "text-[10px] font-tactical uppercase tracking-widest text-emerald-400 mt-0.5";
    }
}

let bannerTimeout = null;
function showBanner(text) {
    const el = document.getElementById("hudAlertBanner");
    el.textContent = text;
    el.style.opacity = "1";
    clearTimeout(bannerTimeout);
    bannerTimeout = setTimeout(() => {
        el.style.opacity = "0";
    }, 3500);
}

function showFloatingFeedback(text, colorHex) {
    const container = document.getElementById("feedbackContainer");
    const div = document.createElement("div");
    div.className = "hit-popup font-tactical font-bold text-sm sm:text-lg px-4 py-1.5 rounded-full bg-slate-950/85 border shadow-xl tracking-wide";
    div.style.color = colorHex;
    div.style.borderColor = colorHex;
    div.textContent = text;
    container.appendChild(div);
    setTimeout(() => div.remove(), 1850);
}

// Base Camp Menu Rendering
let selectedMenuWorld = 0;

function renderMainMenu() {
    document.getElementById("menuCash").textContent = saveData.cash.toLocaleString();
    document.getElementById("menuXp").textContent = saveData.xp.toLocaleString();

    // Render World Cards
    const worldGrid = document.getElementById("worldCardsGrid");
    worldGrid.innerHTML = "";
    WORLDS.forEach((w, idx) => {
        const unlocked = saveData.unlockedWorlds[idx];
        const maxLvl = Math.min(20, saveData.maxLevelByWorld[idx]);
        const beaten = saveData.maxLevelByWorld[idx] > 20;
        const isSelected = selectedMenuWorld === idx;

        const card = document.createElement("div");
        card.className = `rounded-xl p-3.5 border transition cursor-pointer ${
            !unlocked
                ? "bg-slate-950/60 border-slate-800 opacity-60"
                : isSelected
                    ? "bg-slate-800/90 border-amber-400 shadow-lg"
                    : "bg-slate-950/80 border-slate-700 hover:border-slate-500"
        }`;

        card.innerHTML = `
            <div class="flex items-center justify-between mb-1">
                <span class="text-[10px] font-tactical font-bold uppercase px-2 py-0.5 rounded ${
                    unlocked ? "bg-amber-500/20 text-amber-400" : "bg-slate-800 text-slate-500"
                }">WORLD ${idx + 1}</span>
                <span class="text-xs font-tactical ${beaten ? "text-emerald-400" : "text-slate-400"}">
                    ${!unlocked ? "🔒 LOCKED" : beaten ? "★ CLEARED" : `LVL ${maxLvl}/20`}
                </span>
            </div>
            <div class="font-tactical font-bold text-sm sm:text-base text-white truncate">${w.name}</div>
            <div class="text-[11px] text-slate-400 mt-0.5">${w.region}</div>
        `;

        card.onclick = () => {
            if (!unlocked) {
                showBanner(`Complete Level 20 in ${WORLDS[idx - 1].name} to unlock!`);
                return;
            }
            selectedMenuWorld = idx;
            renderMainMenu();
        };
        worldGrid.appendChild(card);
    });

    // Render Selected World Info & Levels 1-20
    const curW = WORLDS[selectedMenuWorld];
    document.getElementById("selectedWorldTitle").textContent = `World ${curW.id + 1}: ${curW.name}`;
    document.getElementById("selectedWorldDesc").textContent = curW.desc;

    const maxUnlockedLvl = saveData.maxLevelByWorld[selectedMenuWorld];
    const nextPlayable = Math.min(20, maxUnlockedLvl);
    const btnContinue = document.getElementById("btnContinueNextLevel");
    btnContinue.textContent = `▶ HUNT LEVEL ${nextPlayable} (${SPECIES_DB[getLevelConfig(selectedMenuWorld, nextPlayable).targetSpeciesKey].name.toUpperCase()})`;
    btnContinue.onclick = () => startExpedition(selectedMenuWorld, nextPlayable);

    const lvlGrid = document.getElementById("levelButtonsGrid");
    lvlGrid.innerHTML = "";
    for (let l = 1; l <= 20; l++) {
        const isUnlocked = l <= maxUnlockedLvl;
        const isCleared = l < maxUnlockedLvl;
        const isCurrent = l === nextPlayable;
        const cfg = getLevelConfig(selectedMenuWorld, l);
        const specName = SPECIES_DB[cfg.targetSpeciesKey].name.split(" ").pop();

        const btn = document.createElement("button");
        btn.disabled = !isUnlocked;
        btn.className = `rounded-xl p-2 border text-center transition flex flex-col items-center justify-center ${
            !isUnlocked
                ? "bg-slate-950/50 border-slate-800/80 text-slate-600 cursor-not-allowed"
                : isCurrent
                    ? "bg-amber-500/20 border-amber-400 text-amber-300 hover:bg-amber-500/30 cursor-pointer shadow"
                    : "bg-slate-900 border-slate-700 text-slate-200 hover:border-emerald-400 cursor-pointer"
        }`;
        btn.innerHTML = `
            <div class="font-tactical font-bold text-sm">${l === 20 ? "👑 20" : l}</div>
            <div class="text-[9px] truncate w-full ${isCleared ? "text-emerald-400" : "text-slate-400"}">
                ${!isUnlocked ? "🔒" : specName}
            </div>
        `;
        btn.onclick = () => startExpedition(selectedMenuWorld, l);
        lvlGrid.appendChild(btn);
    }

    renderArmory();
}

function renderArmory() {
    const grid = document.getElementById("armoryGrid");
    grid.innerHTML = "";

    GUNS_DB.forEach(baseGun => {
        const owned = !!saveData.ownedGuns[baseGun.id];
        const equipped = saveData.equippedGun === baseGun.id;
        const stats = getGunStats(baseGun.id);

        const card = document.createElement("div");
        card.className = `rounded-xl p-4 border flex flex-col justify-between gap-3 ${
            equipped
                ? "bg-slate-800/80 border-amber-400 shadow-lg"
                : owned
                    ? "bg-slate-950/80 border-slate-700"
                    : "bg-slate-950/50 border-slate-800"
        }`;

        const upgCostBase = Math.round(140 + baseGun.cost * 0.14);

        card.innerHTML = `
            <div>
                <div class="flex items-start justify-between gap-2">
                    <div>
                        <div class="font-tactical font-bold text-base text-white">${baseGun.name}</div>
                        <div class="text-xs text-amber-400 font-medium">${baseGun.caliber} • Mag: ${baseGun.magSize}</div>
                    </div>
                    ${
                        equipped
                            ? `<span class="px-2.5 py-1 rounded-lg text-xs font-tactical font-bold bg-amber-500 text-slate-950">EQUIPPED</span>`
                            : owned
                                ? `<button data-equip="${baseGun.id}" class="px-3 py-1 rounded-lg text-xs font-tactical font-bold bg-slate-700 hover:bg-amber-500 hover:text-slate-950 text-white transition cursor-pointer">EQUIP</button>`
                                : `<button data-buy="${baseGun.id}" class="px-3 py-1 rounded-lg text-xs font-tactical font-bold bg-emerald-600 hover:bg-emerald-500 text-white transition cursor-pointer">BUY $${baseGun.cost}</button>`
                    }
                </div>
                <p class="text-xs text-slate-400 mt-1.5">${baseGun.desc}</p>

                <!-- Stats Bars -->
                <div class="grid grid-cols-2 gap-2 mt-3 text-[11px]">
                    <div class="bg-slate-900/90 p-2 rounded-lg border border-slate-800">
                        <div class="flex justify-between text-slate-300"><span>Stopping Power</span><strong class="text-amber-400">${stats.power}</strong></div>
                        ${owned ? `<button data-upg="${baseGun.id}:power" class="mt-1.5 w-full py-1 rounded bg-slate-800 hover:bg-slate-700 text-[10px] font-tactical text-emerald-400 cursor-pointer">
                            ${stats.upgrades.power >= 4 ? "MAXED" : `+ POWER ($${upgCostBase * (stats.upgrades.power + 1)})`}
                        </button>` : ""}
                    </div>
                    <div class="bg-slate-900/90 p-2 rounded-lg border border-slate-800">
                        <div class="flex justify-between text-slate-300"><span>Scope Zoom</span><strong class="text-cyan-400">${stats.zoomMax}x</strong></div>
                        ${owned ? `<button data-upg="${baseGun.id}:zoom" class="mt-1.5 w-full py-1 rounded bg-slate-800 hover:bg-slate-700 text-[10px] font-tactical text-emerald-400 cursor-pointer">
                            ${stats.upgrades.zoom >= 4 ? "MAXED" : `+ OPTICS ($${upgCostBase * (stats.upgrades.zoom + 1)})`}
                        </button>` : ""}
                    </div>
                    <div class="bg-slate-900/90 p-2 rounded-lg border border-slate-800">
                        <div class="flex justify-between text-slate-300"><span>Stability</span><strong class="text-emerald-400">${Math.round(stats.stability * 100)}%</strong></div>
                        ${owned ? `<button data-upg="${baseGun.id}:stability" class="mt-1.5 w-full py-1 rounded bg-slate-800 hover:bg-slate-700 text-[10px] font-tactical text-emerald-400 cursor-pointer">
                            ${stats.upgrades.stability >= 4 ? "MAXED" : `+ BIPOD ($${upgCostBase * (stats.upgrades.stability + 1)})`}
                        </button>` : ""}
                    </div>
                    <div class="bg-slate-900/90 p-2 rounded-lg border border-slate-800">
                        <div class="flex justify-between text-slate-300"><span>Reload Time</span><strong class="text-purple-400">${stats.reloadTime}s</strong></div>
                        ${owned ? `<button data-upg="${baseGun.id}:reload" class="mt-1.5 w-full py-1 rounded bg-slate-800 hover:bg-slate-700 text-[10px] font-tactical text-emerald-400 cursor-pointer">
                            ${stats.upgrades.reload >= 4 ? "MAXED" : `+ BOLT ($${upgCostBase * (stats.upgrades.reload + 1)})`}
                        </button>` : ""}
                    </div>
                </div>
            </div>
        `;
        grid.appendChild(card);
    });

    // Attach Armory Button Handlers
    grid.querySelectorAll("[data-buy]").forEach(btn => {
        btn.onclick = () => {
            const id = btn.getAttribute("data-buy");
            const gun = GUNS_DB.find(g => g.id === id);
            if (saveData.cash >= gun.cost) {
                saveData.cash -= gun.cost;
                saveData.ownedGuns[id] = { power: 0, zoom: 0, stability: 0, reload: 0 };
                saveData.equippedGun = id;
                writeSave();
                audio.playReload();
                renderMainMenu();
            } else {
                showFloatingFeedback("INSUFFICIENT HUNTER FUNDS", "#f87171");
            }
        };
    });

    grid.querySelectorAll("[data-equip]").forEach(btn => {
        btn.onclick = () => {
            saveData.equippedGun = btn.getAttribute("data-equip");
            writeSave();
            audio.playReload();
            renderMainMenu();
        };
    });

    grid.querySelectorAll("[data-upg]").forEach(btn => {
        btn.onclick = () => {
            const [gunId, stat] = btn.getAttribute("data-upg").split(":");
            const baseGun = GUNS_DB.find(g => g.id === gunId);
            const curLvl = saveData.ownedGuns[gunId][stat];
            if (curLvl >= 4) return;
            const cost = Math.round(140 + baseGun.cost * 0.14) * (curLvl + 1);
            if (saveData.cash >= cost) {
                saveData.cash -= cost;
                saveData.ownedGuns[gunId][stat]++;
                writeSave();
                audio.playHitMarker(true);
                renderMainMenu();
            } else {
                showFloatingFeedback("NOT ENOUGH CASH FOR UPGRADE", "#f87171");
            }
        };
    });
}

// ----------------------------------------------------------------------------
// 8. INPUT LISTENERS (MOUSE, KEYBOARD & TOUCH)
// ----------------------------------------------------------------------------
let isDragging = false;
let lastPointerX = 0;
let lastPointerY = 0;

canvas.addEventListener("contextmenu", e => e.preventDefault());

canvas.addEventListener("mousedown", e => {
    if (!gameState.active) return;
    audio.init();
    if (e.button === 2) {
        setScoped(!gameState.isScoped);
    } else if (e.button === 0) {
        isDragging = true;
        lastPointerX = e.clientX;
        lastPointerY = e.clientY;
        // Fire if scoped or quick-click
        if (gameState.isScoped) {
            fireShot();
        }
    }
});

window.addEventListener("mouseup", e => {
    if (e.button === 0) {
        isDragging = false;
    }
});

canvas.addEventListener("dblclick", e => {
    if (gameState.active && !gameState.isScoped) {
        fireShot();
    }
});

window.addEventListener("mousemove", e => {
    if (!gameState.active) return;
    const sensitivity = (gameState.isScoped ? 0.0014 / (gameState.currentZoom * 0.45) : 0.0026);
    if (document.pointerLockElement === canvas) {
        gameState.yaw -= e.movementX * sensitivity;
        gameState.pitch -= e.movementY * sensitivity;
    } else if (isDragging || gameState.isScoped) {
        const dx = e.movementX || (e.clientX - lastPointerX);
        const dy = e.movementY || (e.clientY - lastPointerY);
        gameState.yaw -= dx * sensitivity;
        gameState.pitch -= dy * sensitivity;
    }
    lastPointerX = e.clientX;
    lastPointerY = e.clientY;
    gameState.pitch = Math.max(-0.45, Math.min(0.65, gameState.pitch));
});

window.addEventListener("wheel", e => {
    if (!gameState.active || !gameState.isScoped) return;
    cycleZoom(e.deltaY < 0 ? 1 : -1);
});

window.addEventListener("keydown", e => {
    if (!gameState.active) return;
    if (e.code === "ShiftLeft" || e.code === "ShiftRight" || e.code === "Space") {
        if (gameState.isScoped && !gameState.breathExhausted) {
            gameState.holdingBreath = true;
        }
    } else if (e.code === "KeyE") {
        setScoped(!gameState.isScoped);
    } else if (e.code === "KeyZ") {
        cycleZoom(1);
    } else if (e.code === "KeyR") {
        reloadWeapon();
    } else if (e.code === "KeyF") {
        fireShot();
    }
});

window.addEventListener("keyup", e => {
    if (e.code === "ShiftLeft" || e.code === "ShiftRight" || e.code === "Space") {
        gameState.holdingBreath = false;
    }
});

// Touch Controls for Mobile/Tablet
canvas.addEventListener("touchstart", e => {
    if (!gameState.active) return;
    audio.init();
    if (e.touches.length === 1) {
        isDragging = true;
        lastPointerX = e.touches[0].clientX;
        lastPointerY = e.touches[0].clientY;
    }
}, { passive: true });

canvas.addEventListener("touchmove", e => {
    if (!gameState.active || !isDragging || e.touches.length !== 1) return;
    const sensitivity = gameState.isScoped ? 0.0018 / (gameState.currentZoom * 0.4) : 0.0032;
    const dx = e.touches[0].clientX - lastPointerX;
    const dy = e.touches[0].clientY - lastPointerY;
    gameState.yaw -= dx * sensitivity;
    gameState.pitch -= dy * sensitivity;
    gameState.pitch = Math.max(-0.45, Math.min(0.65, gameState.pitch));
    lastPointerX = e.touches[0].clientX;
    lastPointerY = e.touches[0].clientY;
}, { passive: true });

canvas.addEventListener("touchend", () => {
    isDragging = false;
});

// HUD & Menu Button Wiring
document.getElementById("btnToggleScope").onclick = () => setScoped(!gameState.isScoped);
document.getElementById("btnZoomCycle").onclick = () => cycleZoom(1);
document.getElementById("btnReload").onclick = () => reloadWeapon();
document.getElementById("btnMobileFire").onclick = () => fireShot();

const btnHoldBreath = document.getElementById("btnHoldBreath");
btnHoldBreath.addEventListener("mousedown", () => { if (!gameState.isScoped) setScoped(true); gameState.holdingBreath = true; });
btnHoldBreath.addEventListener("mouseup", () => { gameState.holdingBreath = false; });
btnHoldBreath.addEventListener("touchstart", (e) => { e.preventDefault(); if (!gameState.isScoped) setScoped(true); gameState.holdingBreath = true; });
btnHoldBreath.addEventListener("touchend", () => { gameState.holdingBreath = false; });

document.getElementById("btnPauseMenu").onclick = () => {
    gameState.active = false;
    setScoped(false);
    document.getElementById("gameHud").classList.add("hidden");
    document.getElementById("dangerOverlay").classList.add("hidden");
    selectedMenuWorld = gameState.worldIdx;
    renderMainMenu();
    document.getElementById("mainMenuOverlay").classList.remove("hidden");
};

document.getElementById("btnEndBackCamp").onclick = () => {
    document.getElementById("roundEndModal").classList.add("hidden");
    document.getElementById("roundEndModal").classList.remove("flex");
    document.getElementById("gameHud").classList.add("hidden");
    selectedMenuWorld = gameState.worldIdx;
    renderMainMenu();
    document.getElementById("mainMenuOverlay").classList.remove("hidden");
};

// Menu Tabs
function switchMenuTab(tabName) {
    ["Expeditions", "Armory", "Ballistics"].forEach(name => {
        const btn = document.getElementById(`tab${name}`);
        const content = document.getElementById(`content${name}`);
        if (name === tabName) {
            btn.className = "px-4 py-2.5 font-tactical text-xs sm:text-sm font-bold uppercase tracking-wider rounded-t-lg border-t border-x border-slate-700 bg-slate-900 text-amber-400 cursor-pointer";
            content.classList.remove("hidden");
        } else {
            btn.className = "px-4 py-2.5 font-tactical text-xs sm:text-sm font-bold uppercase tracking-wider rounded-t-lg border-t border-x border-transparent text-slate-400 hover:text-slate-200 cursor-pointer";
            content.classList.add("hidden");
        }
    });
}
document.getElementById("tabExpeditions").onclick = () => switchMenuTab("Expeditions");
document.getElementById("tabArmory").onclick = () => switchMenuTab("Armory");
document.getElementById("tabBallistics").onclick = () => switchMenuTab("Ballistics");

document.getElementById("btnResetProgress").onclick = () => {
    localStorage.removeItem(SAVE_KEY);
    location.reload();
};

window.addEventListener("resize", () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});

// ----------------------------------------------------------------------------
// 9. MAIN 60FPS SIMULATION & RENDER LOOP
// ----------------------------------------------------------------------------
let lastTime = performance.now();

function animate(now) {
    requestAnimationFrame(animate);
    const dt = Math.min(0.05, (now - lastTime) / 1000);
    lastTime = now;

    // Gentle tree sway & weather drift even behind menus
    swayingTrees.forEach(t => {
        t.mesh.rotation.z = Math.sin(now * 0.0012 + t.phase) * 0.018 * gameState.windSpeed;
    });

    if (weatherParticles) {
        const pos = weatherParticles.geometry.attributes.position;
        const isSnow = WORLDS[gameState.worldIdx].weather === "snow";
        for (let i = 0; i < pos.count; i++) {
            let py = pos.getY(i) - (isSnow ? 4.5 : 0.6) * dt;
            let px = pos.getX(i) + gameState.windVec.x * gameState.windSpeed * 0.8 * dt;
            if (py < 0) py = 42;
            if (px > 80) px = -80;
            if (px < -80) px = 80;
            pos.setY(i, py);
            pos.setX(i, px);
        }
        pos.needsUpdate = true;
    }

    if (gameState.active) {
        const gun = getGunStats(saveData.equippedGun);

        // Reload countdown
        if (gameState.isReloading) {
            gameState.reloadTimer -= dt;
            if (gameState.reloadTimer <= 0) {
                gameState.isReloading = false;
                gameState.ammo = gun.magSize;
                updateHud();
            }
        }

        // Hold-Breath Stamina & Scope Sway
        if (gameState.holdingBreath && gameState.isScoped && !gameState.breathExhausted) {
            gameState.breathStamina -= dt * 26;
            audio.heartbeatTimer -= dt;
            if (audio.heartbeatTimer <= 0) {
                audio.playHeartbeat();
                audio.heartbeatTimer = 0.72;
            }
            if (gameState.breathStamina <= 0) {
                gameState.breathStamina = 0;
                gameState.breathExhausted = true;
                gameState.holdingBreath = false;
            }
        } else {
            gameState.breathStamina = Math.min(100, gameState.breathStamina + dt * 18);
            if (gameState.breathStamina > 35) {
                gameState.breathExhausted = false;
            }
        }

        // Update Scope Breath UI
        const breathBar = document.getElementById("breathBar");
        const breathText = document.getElementById("breathStatusText");
        if (breathBar) {
            breathBar.style.width = `${gameState.breathStamina}%`;
            if (gameState.breathExhausted) {
                breathBar.className = "h-full bg-red-500 transition-all duration-75";
                breathText.textContent = "RECOVERING BREATH...";
            } else if (gameState.holdingBreath) {
                breathBar.className = "h-full bg-cyan-400 transition-all duration-75";
                breathText.textContent = "STEADY AIM ACTIVE";
            } else {
                breathBar.className = "h-full bg-emerald-400 transition-all duration-75";
                breathText.textContent = "HOLD [SHIFT] STEADY";
            }
        }

        // Calculate Scope Sway & Recoil Recovery
        gameState.swayTime += dt * (gameState.breathExhausted ? 4.2 : 1.7);
        const baseSwayAmp = gameState.isScoped ? (1.05 - gun.stability) * 0.016 : 0.001;
        const swayMult = gameState.holdingBreath ? 0.06 : (gameState.breathExhausted ? 1.85 : 1.0);
        const swayYaw = Math.sin(gameState.swayTime) * baseSwayAmp * swayMult;
        const swayPitch = Math.cos(gameState.swayTime * 1.4) * baseSwayAmp * 0.75 * swayMult;

        gameState.recoilPitch *= Math.pow(0.04, dt);

        // Apply Euler orientation to First-Person Camera
        const euler = new THREE.Euler(
            gameState.pitch + swayPitch + gameState.recoilPitch,
            gameState.yaw + swayYaw,
            0,
            "YXZ"
        );
        camera.quaternion.setFromEuler(euler);

        // Digital Laser Rangefinder when Scoped
        if (gameState.isScoped) {
            const fwd = new THREE.Vector3(0, 0, -1).applyQuaternion(camera.quaternion);
            raycaster.set(camera.position, fwd);
            const hitMeshes = [];
            gameState.animals.forEach(a => {
                if (a.alive) a.hitboxes.forEach(hb => hitMeshes.push(hb.mesh));
            });
            const hits = raycaster.intersectObjects(hitMeshes, false);
            const rngEl = document.getElementById("scopeRangeVal");
            if (hits.length > 0) {
                rngEl.textContent = Math.round(hits[0].distance);
                rngEl.className = "text-amber-400 font-bold";
            } else {
                rngEl.textContent = "---";
                rngEl.className = "text-emerald-400";
            }
        }

        // Update Wildlife AI & Check Predator Charges
        let predatorDanger = false;
        for (const animal of gameState.animals) {
            animal.update(dt, camera.position, gameState.windVec);
            if (animal.alive && animal.state === "charge") {
                const d = animal.root.position.distanceTo(camera.position);
                if (d < 38) predatorDanger = true;
                if (d < 4.2) {
                    failLevel(`A charging ${animal.spec.name} reached your blind!`);
                    break;
                }
            }
        }

        const dangerEl = document.getElementById("dangerOverlay");
        if (predatorDanger) {
            dangerEl.classList.remove("hidden");
        } else {
            dangerEl.classList.add("hidden");
        }

        // Fade bullet tracers
        for (let i = gameState.tracers.length - 1; i >= 0; i--) {
            const tr = gameState.tracers[i];
            tr.life -= dt;
            tr.line.material.opacity = Math.max(0, tr.life * 3.2);
            if (tr.life <= 0) {
                fxGroup.remove(tr.line);
                gameState.tracers.splice(i, 1);
            }
        }
    } else {
        // Gentle scenic camera pan while in Camp Menu
        camera.position.set(0, 4.5, 0);
        camera.rotation.set(-0.04, Math.sin(now * 0.00018) * 0.45, 0, "YXZ");
    }

    renderer.render(scene, camera);
}

// ----------------------------------------------------------------------------
// 10. INITIALIZATION ON WINDOW LOAD
// ----------------------------------------------------------------------------
window.onload = function () {
    loadSave();
    buildEnvironment(0);
    renderMainMenu();
    animate(performance.now());
};
</script>
</body>
</html>
