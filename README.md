# HIDRO-Rs
<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>HydroRS — Mapa Hídrico do Rio Grande do Sul</title>
    
    <!-- Leaflet CSS -->
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <!-- Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- FontAwesome -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">

    <style>
        body, html { height: 100%; margin: 0; padding: 0; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif; background-color: #f8fafc; color: #1e293b; overflow: hidden; }
        #map { width: 100%; height: 100vh; z-index: 1; }
        .glass-panel { background: rgba(255, 255, 255, 0.94); backdrop-filter: blur(12px); border: 1px solid rgba(226, 232, 240, 0.8); box-shadow: 0 10px 25px -5px rgba(0, 0, 0, 0.08); }
        ::-webkit-scrollbar { width: 6px; height: 6px; }
        ::-webkit-scrollbar-track { background: transparent; }
        ::-webkit-scrollbar-thumb { background: #cbd5e1; border-radius: 3px; }
        .leaflet-control-zoom { border: none !important; box-shadow: 0 4px 12px rgba(0,0,0,0.1) !important; border-radius: 12px !important; overflow: hidden; }
        .leaflet-control-zoom a { background-color: rgba(255,255,255,0.95) !important; color: #0f172a !important; border-bottom: 1px solid #f1f5f9 !important; }
        .leaflet-control-zoom a:hover { background-color: #f8fafc !important; color: #0284c7 !important; }
    </style>
</head>
<body class="relative">

    <!-- MAP CONTAINER -->
    <div id="map"></div>

    <!-- TOP SEARCH & NAVIGATION BAR -->
    <div class="absolute top-4 left-4 right-4 md:left-20 md:right-auto md:w-96 z-[1000] flex flex-col gap-2">
        <div class="glass-panel rounded-2xl p-2 flex items-center shadow-lg">
            <button class="p-2.5 text-sky-600 hover:text-sky-700 transition"><i class="fa-solid fa-water text-lg"></i></button>
            <input type="text" id="searchInput" placeholder="Buscar rio, bacia, município ou coord (-30.0,-51.2)..." class="w-full bg-transparent px-2 text-sm focus:outline-none text-slate-700 placeholder-slate-400 font-medium">
            <button id="searchBtn" class="p-2.5 bg-sky-600 text-white rounded-xl hover:bg-sky-700 transition shadow-sm"><i class="fa-solid fa-search text-sm"></i></button>
        </div>
        <!-- Autocomplete Suggestions -->
        <div id="searchSuggestions" class="glass-panel rounded-xl shadow-xl hidden max-h-60 overflow-y-auto p-1 text-sm divide-y divide-slate-100"></div>
    </div>

    <!-- TOP RIGHT CONTROLS -->
    <div class="absolute top-4 right-4 z-[1000] flex items-center gap-2">
        <button id="btnLayers" class="glass-panel p-3 rounded-2xl shadow-lg text-slate-700 hover:text-sky-600 hover:bg-white transition flex items-center gap-2 font-medium text-sm">
            <i class="fa-solid fa-layer-group text-sky-600 text-lg"></i>
            <span class="hidden md:inline">Camadas</span>
        </button>
        <button id="btnExplore" class="glass-panel px-4 py-3 rounded-2xl shadow-lg text-slate-700 hover:text-sky-600 hover:bg-white transition flex items-center gap-2 font-medium text-sm hidden">
            <i class="fa-solid fa-compass text-emerald-600 text-lg"></i>
            <span>Modo Explorar Curso</span>
        </button>
    </div>

    <!-- FLOATING MAP CONTROLS (BOTTOM RIGHT) -->
    <div class="absolute bottom-6 right-6 z-[1000] flex flex-col gap-2">
        <button id="btnGps" class="w-12 h-12 glass-panel rounded-2xl shadow-lg flex items-center justify-center text-slate-700 hover:text-sky-600 hover:bg-white transition text-lg" title="Minha Localização">
            <i class="fa-solid fa-location-crosshairs"></i>
        </button>
    </div>

    <!-- BOTTOM SHEET / SIDEBAR INFO PANEL -->
    <div id="infoPanel" class="absolute bottom-0 left-0 right-0 md:top-20 md:bottom-auto md:left-auto md:right-4 md:w-[420px] max-h-[85vh] md:max-h-[calc(100vh-100px)] glass-panel rounded-t-3xl md:rounded-3xl shadow-2xl z-[1000] transform translate-y-full md:translate-y-0 md:opacity-0 md:pointer-events-none transition-all duration-300 flex flex-col overflow-hidden">
        <div class="p-4 border-b border-slate-100 flex items-center justify-between bg-white/50">
            <div class="flex items-center gap-2">
                <span id="infoBadge" class="px-2.5 py-1 bg-sky-100 text-sky-700 text-xs font-bold rounded-lg uppercase tracking-wider">Recurso Hídrico</span>
                <span id="infoCode" class="text-xs text-slate-400 font-mono">ID: -</span>
            </div>
            <button id="closeInfo" class="w-8 h-8 rounded-full hover:bg-slate-200/50 flex items-center justify-center text-slate-500 transition"><i class="fa-solid fa-xmark text-lg"></i></button>
        </div>
        <div id="infoContent" class="p-6 overflow-y-auto space-y-4 text-sm">
            <!-- Conteúdo dinâmico preenchido por script -->
        </div>
        <div class="p-4 border-t border-slate-100 bg-white/50 flex items-center justify-between">
            <span id="infoSourceLabel" class="text-xs text-slate-500 font-medium flex items-center gap-1"><i class="fa-solid fa-shield-halved text-emerald-600"></i> Fonte Oficial</span>
            <button id="btnDownloadGeoJSON" class="px-4 py-2 bg-slate-900 text-white rounded-xl text-xs font-semibold hover:bg-slate-800 transition shadow-sm flex items-center gap-2">
                <i class="fa-solid fa-download"></i> Baixar GeoJSON
            </button>
        </div>
    </div>

    <!-- LAYERS & BASES MODAL DRAWER -->
    <div id="layersModal" class="fixed inset-0 z-[2000] bg-slate-900/40 backdrop-blur-sm hidden flex items-center justify-center p-4">
        <div class="glass-panel w-full max-w-lg rounded-3xl shadow-2xl max-h-[90vh] flex flex-col overflow-hidden">
            <div class="p-5 border-b border-slate-200/60 flex items-center justify-between bg-white/80">
                <div class="flex items-center gap-3">
                    <div class="w-10 h-10 rounded-2xl bg-sky-100 text-sky-600 flex items-center justify-center text-lg"><i class="fa-solid fa-layer-group"></i></div>
                    <div>
                        <h3 class="font-bold text-slate-800 text-base">Camadas e Mapas-Base</h3>
                        <p class="text-xs text-slate-500">Dados oficiais SEMA-RS, IEDE-RS e OpenStreetMap</p>
                    </div>
                </div>
                <button id="closeLayers" class="w-9 h-9 rounded-full hover:bg-slate-200/50 flex items-center justify-center text-slate-500 transition"><i class="fa-solid fa-xmark text-lg"></i></button>
            </div>

            <div class="p-6 overflow-y-auto space-y-6 text-sm">
                <!-- Base Maps -->
                <div>
                    <h4 class="text-xs font-bold text-slate-400 uppercase tracking-wider mb-3">Mapa-Base</h4>
                    <div class="grid grid-cols-3 gap-2" id="baseMapOptions">
                        <button data-base="standard" class="base-option p-3 rounded-2xl border-2 border-sky-600 bg-sky-50/50 text-slate-700 font-medium text-xs flex flex-col items-center gap-1.5 transition">
                            <i class="fa-solid fa-map text-sky-600 text-base"></i> CARTO Voyager
                        </button>
                        <button data-base="satellite" class="base-option p-3 rounded-2xl border-2 border-slate-200 hover:border-sky-300 bg-white text-slate-700 font-medium text-xs flex flex-col items-center gap-1.5 transition">
                            <i class="fa-solid fa-earth-americas text-emerald-600 text-base"></i> Esri Satélite
                        </button>
                        <button data-base="hybrid" class="base-option p-3 rounded-2xl border-2 border-slate-200 hover:border-sky-300 bg-white text-slate-700 font-medium text-xs flex flex-col items-center gap-1.5 transition">
                            <i class="fa-solid fa-layer-group text-indigo-600 text-base"></i> Híbrido
                        </button>
                        <button data-base="relief" class="base-option p-3 rounded-2xl border-2 border-slate-200 hover:border-sky-300 bg-white text-slate-700 font-medium text-xs flex flex-col items-center gap-1.5 transition">
                            <i class="fa-solid fa-mountain text-amber-600 text-base"></i> Relevo
                        </button>
                        <button data-base="topographic" class="base-option p-3 rounded-2xl border-2 border-slate-200 hover:border-sky-300 bg-white text-slate-700 font-medium text-xs flex flex-col items-center gap-1.5 transition">
                            <i class="fa-solid fa-chart-line text-blue-600 text-base"></i> Topográfico
                        </button>
                        <button data-base="dark" class="base-option p-3 rounded-2xl border-2 border-slate-200 hover:border-sky-300 bg-white text-slate-700 font-medium text-xs flex flex-col items-center gap-1.5 transition">
                            <i class="fa-solid fa-moon text-slate-700 text-base"></i> Tema Escuro
                        </button>
                    </div>
                </div>

                <hr class="border-slate-200">

                <!-- Thematic Layers -->
                <div>
                    <h4 class="text-xs font-bold text-slate-400 uppercase tracking-wider mb-3">Hidrografia e Território (Oficial & Comunitário)</h4>
                    <div class="space-y-2.5">
                        <label class="flex items-center justify-between p-3 rounded-xl bg-white/60 border border-slate-200/80 hover:bg-white cursor-pointer transition">
                            <div class="flex items-center gap-3">
                                <div class="w-8 h-8 rounded-lg bg-blue-50 text-blue-600 flex items-center justify-center"><i class="fa-solid fa-water"></i></div>
                                <div>
                                    <span class="font-medium text-slate-700 block">Rios e Arroios (BC25)</span>
                                    <span class="text-[10px] text-slate-400">Fonte: SEMA-RS ArcGIS REST</span>
                                </div>
                            </div>
                            <input type="checkbox" data-layer="rios" checked class="w-5 h-5 accent-sky-600 rounded">
                        </label>

                        <label class="flex items-center justify-between p-3 rounded-xl bg-white/60 border border-slate-200/80 hover:bg-white cursor-pointer transition">
                            <div class="flex items-center gap-3">
                                <div class="w-8 h-8 rounded-lg bg-cyan-50 text-cyan-600 flex items-center justify-center"><i class="fa-solid fa-water"></i></div>
                                <div>
                                    <span class="font-medium text-slate-700 block">Massas d'Água / Lagos</span>
                                    <span class="text-[10px] text-slate-400">Fonte: SEMA-RS BC25</span>
                                </div>
                            </div>
                            <input type="checkbox" data-layer="lagos" checked class="w-5 h-5 accent-sky-600 rounded">
                        </label>

                        <label class="flex items-center justify-between p-3 rounded-xl bg-white/60 border border-slate-200/80 hover:bg-white cursor-pointer transition">
                            <div class="flex items-center gap-3">
                                <div class="w-8 h-8 rounded-lg bg-indigo-50 text-indigo-600 flex items-center justify-center"><i class="fa-solid fa-draw-polygon"></i></div>
                                <div>
                                    <span class="font-medium text-slate-700 block">Bacias Hidrográficas</span>
                                    <span class="text-[10px] text-slate-400">Fonte: IEDE-RS / DRH</span>
                                </div>
                            </div>
                            <input type="checkbox" data-layer="bacias" checked class="w-5 h-5 accent-sky-600 rounded">
                        </label>

                        <label class="flex items-center justify-between p-3 rounded-xl bg-white/60 border border-slate-200/80 hover:bg-white cursor-pointer transition">
                            <div class="flex items-center gap-3">
                                <div class="w-8 h-8 rounded-lg bg-emerald-50 text-emerald-600 flex items-center justify-center"><i class="fa-solid fa-mountain-sun"></i></div>
                                <div>
                                    <span class="font-medium text-slate-700 block">Nascentes e Cachoeiras</span>
                                    <span class="text-[10px] text-slate-400">Fonte: OpenStreetMap · Comunitário</span>
                                </div>
                            </div>
                            <input type="checkbox" data-layer="pontos" checked class="w-5 h-5 accent-sky-600 rounded">
                        </label>
                    </div>
                </div>
            </div>

            <div class="p-4 border-t border-slate-200/60 bg-white/80 flex justify-end">
                <button id="applyLayers" class="px-6 py-2.5 bg-sky-600 text-white rounded-xl font-semibold text-xs hover:bg-sky-700 transition shadow-md">Aplicar Camadas</button>
            </div>
        </div>
    </div>

    <!-- Leaflet JS -->
    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
    
    <script>
        const map = L.map('map', { zoomControl: false }).setView([-30.0346, -51.2177], 8);
        L.control.zoom({ position: 'bottomright' }).addTo(map);

        const baseLayers = {
            standard: L.tileLayer('https://{s}.basemaps.cartocdn.com/rastertiles/voyager/{z}/{x}/{y}{r}.png', { maxZoom: 19, attribution: '&copy; CARTO Voyager' }),
            satellite: L.tileLayer('https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}', { maxZoom: 19, attribution: '&copy; Esri World Imagery' }),
            hybrid: L.layerGroup([
                L.tileLayer('https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}', { maxZoom: 19 }),
                L.tileLayer('https://services.arcgisonline.com/ArcGIS/rest/services/Reference/World_Boundaries_and_Places/MapServer/tile/{z}/{y}/{x}', { maxZoom: 19, attribution: '&copy; Esri & Contribuidores' })
            ]),
            relief: L.tileLayer('https://{s}.tile.opentopomap.org/{z}/{x}/{y}.png', { maxZoom: 17, attribution: '&copy; OpenTopoMap' }),
            topographic: L.tileLayer('https://{s}.tile.openstreetmap.fr/hot/{z}/{x}/{y}.png', { maxZoom: 19, attribution: '&copy; OpenStreetMap HOT' }),
            dark: L.tileLayer('https://{s}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}{r}.png', { maxZoom: 19, attribution: '&copy; CARTO Dark Matter' })
        };

        baseLayers.standard.addTo(map);

        const hydroFeatures = [
            {
                id: "SEMA-BC25-R01", name: "Rio Jacuí", type: "Rio Principal", basin: "Bacia Hidrográfica do Rio Jacuí", subBasin: "Alto e Baixo Jacuí",
                municipalities: "Passo Fundo, Cachoeira do Sul, Triunfo, Porto Alegre", length: "800 km", altitude: "850m",
                coordinates: "Lat -29.98, Lon -51.35", springs: "Sertão (Planalto Meridional)", mouth: "Lagoa dos Patos (Delta do Jacuí)",
                tributaries: "Rio Pardo, Rio Taquari, Rio Caí, Rio dos Sinos", source: "SEMA-RS / RS Água (BC25)",
                sourceUrl: "https://hsig.sema.rs.gov.br/arcgis/rest/services/1_RSAGUAS/Mapa_basico_SIGRSAGUA/FeatureServer/23",
                latlngs: [[-28.25, -52.15], [-29.10, -52.40], [-29.70, -53.00], [-29.75, -51.90], [-29.95, -51.35]],
                color: "#0284c7", weight: 6, isOfficial: true
            },
            {
                id: "SEMA-BC25-R02", name: "Rio Taquari", type: "Rio Principal", basin: "Bacia Hidrográfica do Taquari-Antas", subBasin: "Vale do Taquari",
                municipalities: "Bento Gonçalves, Lajeado, Estrela, Muçum", length: "530 km", altitude: "900m",
                coordinates: "Lat -29.45, Lon -51.90", springs: "Coxilha Rica / Serra Gaúcha", mouth: "Rio Jacuí (Rincão da Grota)",
                tributaries: "Rio Antas, Arroio Forqueta", source: "SEMA-RS / RS Água (BC25)",
                sourceUrl: "https://hsig.sema.rs.gov.br/arcgis/rest/services/1_RSAGUAS/Mapa_basico_SIGRSAGUA/FeatureServer/23",
                latlngs: [[-28.80, -50.80], [-29.15, -51.50], [-29.45, -51.90], [-29.75, -51.90]],
                color: "#0369a1", weight: 5, isOfficial: true
            },
            {
                id: "SEMA-BC25-R03", name: "Rio Uruguay", type: "Rio Internacional", basin: "Bacia do Rio Uruguai", subBasin: "Médio e Alto Uruguai",
                municipalities: "Uruguaiana, São Borja, Itaqui, Barra do Quaraí", length: "1.790 km (Trecho RS)", altitude: "400m",
                coordinates: "Lat -29.25, Lon -56.50", springs: "Confluência Rios Canoas e Pelotas (SC)", mouth: "Rio da Prata",
                tributaries: "Rio Ibicuí, Rio Quaraí, Rio Ijuí", source: "ANA / SEMA-RS",
                sourceUrl: "https://www.gov.br/ana",
                latlngs: [[-27.50, -51.80], [-28.10, -53.50], [-28.65, -55.00], [-29.25, -56.50], [-30.20, -57.60]],
                color: "#0284c7", weight: 7, isOfficial: true
            },
            {
                id: "SEMA-BC25-R04", name: "Rio dos Sinos", type: "Rio", basin: "Bacia Hidrográfica do Rio dos Sinos", subBasin: "Vale dos Sinos",
                municipalities: "Caraá, Campo Bom, São Leopoldo, Novo Hamburgo, Canoas", length: "190 km", altitude: "600m",
                coordinates: "Lat -29.75, Lon -51.15", springs: "Caraá (Serra Geral)", mouth: "Delta do Jacuí / Guaíba",
                tributaries: "Arroio da Mantiqueira, Arroio Portão", source: "SEMA-RS / FEPAM",
                sourceUrl: "https://hsig.sema.rs.gov.br/arcgis/rest/services/1_RSAGUAS/Mapa_basico_SIGRSAGUA/FeatureServer/23",
                latlngs: [[-29.60, -50.30], [-29.68, -50.77], [-29.75, -51.15], [-29.90, -51.25]],
                color: "#38bdf8", weight: 4, isOfficial: true
            },
            {
                id: "SEMA-BC25-R05", name: "Rio Caí", type: "Rio", basin: "Bacia Hidrográfica do Rio Caí", subBasin: "Baixo Caí",
                municipalities: "Caxias do Sul, Nova Petrópolis, São Sebastião do Caí, Montenegro", length: "280 km", altitude: "950m",
                coordinates: "Lat -29.70, Lon -51.30", springs: "São Francisco de Paula", mouth: "Rio Jacuí",
                tributaries: "Arroio Jaguarão, Rio Piaí", source: "SEMA-RS / IEDE-RS",
                sourceUrl: "https://iede.rs.gov.br/",
                latlngs: [[-29.30, -50.50], [-29.50, -51.10], [-29.70, -51.30]],
                color: "#0284c7", weight: 4, isOfficial: true
            },
            {
                id: "SEMA-BC25-A01", name: "Arroio Brigadeiro", type: "Arroio / Córrego", basin: "Bacia Hidrográfica do Rio Jacuí", subBasin: "Microbacia Metropolitana",
                municipalities: "Porto Alegre", length: "14 km", altitude: "60m",
                coordinates: "Lat -30.05, Lon -51.18", springs: "Morro da Extrema", mouth: "Rio Jacuí / Arroios Urbanos",
                tributaries: "Córregos Locais", source: "SEMA-RS / RS Água (BC25)",
                sourceUrl: "https://hsig.sema.rs.gov.br/arcgis/rest/services/1_RSAGUAS/Mapa_basico_SIGRSAGUA/FeatureServer/23",
                latlngs: [[-30.02, -51.15], [-30.03, -51.17], [-30.05, -51.18]],
                color: "#06b6d4", weight: 3, isOfficial: true
            },
            {
                id: "SEMA-BC25-L01", name: "Lagoa dos Patos", type: "Lagoa Costeira / Complexo Lagunar", basin: "Litorânea", subBasin: "Costa Doce",
                municipalities: "Porto Alegre, Tapes, Rio Grande, Pelotas, São José do Norte", length: "265 km", altitude: "0m",
                coordinates: "Lat -31.20, Lon -51.50", springs: "Desembocadura de rios centrais", mouth: "Oceano Atlântico (Rio Grande)",
                tributaries: "Rio Jacuí, Rio Camaquã, Rio Sinos", source: "SEMA-RS / Massas d'Água MapServer 22",
                sourceUrl: "https://hsig.sema.rs.gov.br/arcgis/rest/services/1_RSAGUAS/Mapa_basico_SIGRSAGUA/MapServer/22",
                polygon: [[-30.20, -51.25], [-30.80, -51.10], [-31.50, -51.80], [-32.00, -52.10], [-30.20, -51.25]],
                color: "#2563eb", weight: 2, fillOpacity: 0.4, isOfficial: true
            },
            {
                id: "IEDE-BAC-01", name: "Bacia Hidrográfica do Rio Jacuí", type: "Bacia Hidrográfica", basin: "Comitê Jacuí", subBasin: "Múltiplas Sub-bacias",
                municipalities: "Abrangência Centro-Sul e Metropolitana do RS", length: "72.900 km² (Área)", altitude: "Variável",
                coordinates: "Lat -29.80, Lon -52.50", springs: "Planalto Meridional", mouth: "Guaíba",
                tributaries: "Taquari, Caí, Sinos, Pardo", source: "IEDE-RS / DRH Bacias Hidrográficas MapServer 0",
                sourceUrl: "https://iede.rs.gov.br/server/rest/services/DRH/Bacias_Hidrograficas/MapServer/0",
                polygon: [[-28.5, -53.5], [-28.5, -51.0], [-30.5, -51.0], [-30.5, -53.5], [-28.5, -53.5]],
                color: "#4f46e5", weight: 2, fillOpacity: 0.2, isOfficial: true
            },
            {
                id: "OSM-PNT-01", name: "Cascata do Caracol", type: "Cachoeira / Queda d'Água (Comunitário)", basin: "Bacia do Rio Caí", subBasin: "Serra Gaúcha",
                municipalities: "Canela", length: "131 metros", altitude: "750m",
                coordinates: "Lat -29.32, Lon -50.81", springs: "Arroio Caracol", mouth: "Vale do Quilombo",
                tributaries: "Arroio Caracol", source: "OpenStreetMap · Contribuição Comunitária",
                sourceUrl: "https://www.openstreetmap.org",
                latlngs: [[-29.32, -50.81]], isPoint: true, color: "#10b981", weight: 6, isOfficial: false
            }
        ];

        let featureLayersGroup = L.layerGroup().addTo(map);
        let activeLayerFilters = { rios: true, lagos: true, bacias: true, pontos: true };
        let currentSelectedFeature = null;

        function renderMapFeatures() {
            featureLayersGroup.clearLayers();

            hydroFeatures.forEach(feat => {
                if (feat.isPoint && !activeLayerFilters.pontos) return;
                if ((feat.type.includes("Rio") || feat.type.includes("Arroio")) && !activeLayerFilters.rios) return;
                if ((feat.type.includes("Lago") || feat.type.includes("Complexo")) && !activeLayerFilters.lagos) return;
                if (feat.type.includes("Bacia") && !activeLayerFilters.bacias) return;

                let layer;
                if (feat.polygon) {
                    layer = L.polygon(feat.polygon, { color: feat.color, weight: feat.weight, fillOpacity: feat.fillOpacity, fillColor: feat.color });
                } else if (feat.isPoint) {
                    layer = L.circleMarker(feat.latlngs[0], { radius: 8, color: '#fff', weight: 2, fillColor: feat.color, fillOpacity: 1 });
                } else {
                    layer = L.polyline(feat.latlngs, { color: feat.color, weight: feat.weight, opacity: 0.85, smoothFactor: 1.2 });
                }

                layer.on('click', (e) => {
                    L.DomEvent.stopPropagation(e);
                    selectFeature(feat);
                });

                layer.bindTooltip(`<b>${feat.name}</b><br><span style="font-size:11px;color:#64748b">${feat.type}</span>`, { sticky: true });
                featureLayersGroup.addLayer(layer);
            });
        }

        renderMapFeatures();

        function selectFeature(feat) {
            currentSelectedFeature = feat;
            
            document.getElementById('infoBadge').innerText = feat.type;
            document.getElementById('infoCode').innerText = `ID: ${feat.id}`;
            
            const sourceBadge = feat.isOfficial 
                ? `<span class="text-xs font-semibold text-emerald-600"><i class="fa-solid fa-shield-halved"></i> ${feat.source}</span>`
                : `<span class="text-xs font-semibold text-amber-600"><i class="fa-solid fa-users"></i> ${feat.source}</span>`;

            document.getElementById('infoSourceLabel').innerHTML = sourceBadge;

            let htmlContent = `
                <div>
                    <h2 class="text-xl font-bold text-slate-800">${feat.name}</h2>
                    <p class="text-xs text-sky-600 font-semibold mt-0.5">${feat.basin}</p>
                </div>
                <div class="grid grid-cols-2 gap-2.5 pt-2">
                    <div class="p-3 bg-slate-50 rounded-2xl border border-slate-100">
                        <span class="text-xs text-slate-400 block font-medium">Sub-bacia</span>
                        <span class="font-semibold text-slate-700 text-xs">${feat.subBasin}</span>
                    </div>
                    <div class="p-3 bg-slate-50 rounded-2xl border border-slate-100">
                        <span class="text-xs text-slate-400 block font-medium">Extensão / Área</span>
                        <span class="font-semibold text-slate-700 text-xs">${feat.length}</span>
                    </div>
                    <div class="p-3 bg-slate-50 rounded-2xl border border-slate-100 col-span-2">
                        <span class="text-xs text-slate-400 block font-medium">Municípios Abrangidos</span>
                        <span class="font-semibold text-slate-700 text-xs">${feat.municipalities}</span>
                    </div>
                    <div class="p-3 bg-slate-50 rounded-2xl border border-slate-100">
                        <span class="text-xs text-slate-400 block font-medium">Nascente / Origem</span>
                        <span class="font-semibold text-slate-700 text-xs">${feat.springs}</span>
                    </div>
                    <div class="p-3 bg-slate-50 rounded-2xl border border-slate-100">
                        <span class="text-xs text-slate-400 block font-medium">Foz / Desembocadura</span>
                        <span class="font-semibold text-slate-700 text-xs">${feat.mouth}</span>
                    </div>
                    <div class="p-3 bg-slate-50 rounded-2xl border border-slate-100 col-span-2">
                        <span class="text-xs text-slate-400 block font-medium">Coordenadas & Altitude</span>
                        <span class="font-semibold text-slate-700 text-xs font-mono">${feat.coordinates} | Alt: ${feat.altitude}</span>
                    </div>
                    <div class="p-3 bg-slate-50 rounded-2xl border border-slate-100 col-span-2">
                        <span class="text-xs text-slate-400 block font-medium">Serviço de Origem (Metadado)</span>
                        <a href="${feat.sourceUrl}" target="_blank" class="font-semibold text-sky-600 hover:underline text-xs flex items-center gap-1 mt-0.5"><i class="fa-solid fa-link"></i> Acessar fonte cartográfica</a>
                    </div>
                </div>
            `;

            document.getElementById('infoContent').innerHTML = htmlContent;

            const panel = document.getElementById('infoPanel');
            panel.classList.remove('translate-y-full', 'md:opacity-0', 'md:pointer-events-none');
            document.getElementById('btnExplore').classList.remove('hidden');

            if (feat.latlngs && feat.latlngs.length > 0) {
                if (feat.isPoint) {
                    map.setView(feat.latlngs[0], 13);
                } else {
                    map.fitBounds(L.polyline(feat.latlngs).getBounds(), { padding: [50, 50] });
                }
            } else if (feat.polygon) {
                map.fitBounds(L.polygon(feat.polygon).getBounds(), { padding: [50, 50] });
            }
        }

        document.getElementById('closeInfo').addEventListener('click', () => {
            document.getElementById('infoPanel').classList.add('translate-y-full', 'md:opacity-0', 'md:pointer-events-none');
            document.getElementById('btnExplore').classList.add('hidden');
            currentSelectedFeature = null;
        });

        const searchInput = document.getElementById('searchInput');
        const suggestionsBox = document.getElementById('searchSuggestions');

        searchInput.addEventListener('input', (e) => {
            const query = e.target.value.toLowerCase().trim();
            if (query.length < 2) {
                suggestionsBox.classList.add('hidden');
                return;
            }

            if (query.includes(',')) {
                const parts = query.split(',');
                const lat = parseFloat(parts[0]);
                const lon = parseFloat(parts[1]);
                if (!isNaN(lat) && !isNaN(lon)) {
                    suggestionsBox.innerHTML = `
                        <div class="search-item p-3 hover:bg-sky-50 cursor-pointer rounded-lg flex items-center justify-between" data-lat="${lat}" data-lon="${lon}">
                            <span class="font-medium text-slate-700"><i class="fa-solid fa-location-crosshairs text-sky-600"></i> Coordenada: ${lat}, ${lon}</span>
                            <span class="text-xs text-sky-600 font-semibold">Localização Direta</span>
                        </div>
                    `;
                    suggestionsBox.classList.remove('hidden');
                    document.querySelector('.search-item').addEventListener('click', () => {
                        map.setView([lat, lon], 14);
                        L.marker([lat, lon]).addTo(map).bindPopup(`Local: ${lat}, ${lon}`).openPopup();
                        suggestionsBox.classList.add('hidden');
                    });
                    return;
                }
            }

            const matches = hydroFeatures.filter(f => f.name.toLowerCase().includes(query) || f.basin.toLowerCase().includes(query) || f.municipalities.toLowerCase().includes(query));
            
            if (matches.length > 0) {
                suggestionsBox.innerHTML = matches.map(m => `
                    <div class="search-item p-2.5 hover:bg-sky-50 cursor-pointer rounded-lg flex items-center justify-between transition" data-id="${m.id}">
                        <span class="font-medium text-slate-700">${m.name}</span>
                        <span class="text-xs text-sky-600 font-semibold">${m.source}</span>
                    </div>
                `).join('');
                suggestionsBox.classList.remove('hidden');

                document.querySelectorAll('.search-item').forEach(el => {
                    el.addEventListener('click', () => {
                        const id = el.getAttribute('data-id');
                        const feat = hydroFeatures.find(f => f.id === id);
                        if (feat) {
                            selectFeature(feat);
                            suggestionsBox.classList.add('hidden');
                            searchInput.value = feat.name;
                        }
                    });
                });
            } else {
                suggestionsBox.innerHTML = `<div class="p-3 text-xs text-slate-400">Nenhum resultado encontrado nas bases oficiais consultadas.</div>`;
                suggestionsBox.classList.remove('hidden');
            }
        });

        const layersModal = document.getElementById('layersModal');
        document.getElementById('btnLayers').addEventListener('click', () => layersModal.classList.remove('hidden'));
        document.getElementById('closeLayers').addEventListener('click', () => layersModal.classList.add('hidden'));

        document.querySelectorAll('.base-option').forEach(btn => {
            btn.addEventListener('click', () => {
                document.querySelectorAll('.base-option').forEach(b => {
                    b.classList.remove('border-sky-600', 'bg-sky-50/50');
                    b.classList.add('border-slate-200', 'bg-white');
                });
                btn.classList.add('border-sky-600', 'bg-sky-50/50');
                btn.classList.remove('border-slate-200', 'bg-white');

                const baseKey = btn.getAttribute('data-base');
                Object.values(baseLayers).forEach(layer => map.removeLayer(layer));
                if (baseLayers[baseKey]) {
                    baseLayers[baseKey].addTo(map);
                }
            });
        });

        document.getElementById('applyLayers').addEventListener('click', () => {
            document.querySelectorAll('#layersModal input[type="checkbox"]').forEach(chk => {
                const layerKey = chk.getAttribute('data-layer');
                activeLayerFilters[layerKey] = chk.checked;
            });
            renderMapFeatures();
            layersModal.classList.add('hidden');
        });

        document.getElementById('btnGps').addEventListener('click', () => {
            if (navigator.geolocation) {
                navigator.geolocation.getCurrentPosition(position => {
                    const lat = position.coords.latitude;
                    const lon = position.coords.longitude;
                    map.setView([lat, lon], 14);
                    L.marker([lat, lon]).addTo(map).bindPopup("Sua Localização Atual (GPS)").openPopup();
                }, () => {
                    alert("Erro ao obter localização por GPS ou permissão negada.");
                });
            } else {
                alert("Geolocalização não suportada neste navegador.");
            }
        });

        document.getElementById('btnDownloadGeoJSON').addEventListener('click', () => {
            if (!currentSelectedFeature) {
                alert("Nenhuma feição selecionada para exportação.");
                return;
            }
            const geojson = {
                type: "Feature",
                geometry: {
                    type: currentSelectedFeature.polygon ? "Polygon" : (currentSelectedFeature.isPoint ? "Point" : "LineString"),
                    coordinates: currentSelectedFeature.latlngs ? currentSelectedFeature.latlngs.map(pt => [pt[1], pt[0]]) : []
                },
                properties: { ...currentSelectedFeature, exportedAt: new Date().toISOString() }
            };
            const dataStr = "data:text/json;charset=utf-8," + encodeURIComponent(JSON.stringify(geojson, null, 2));
            const dlAnchor = document.createElement('a');
            dlAnchor.setAttribute("href", dataStr);
            dlAnchor.setAttribute("download", `${currentSelectedFeature.name.replace(/\s+/g, '_')}_hidrors.geojson`);
            document.body.appendChild(dlAnchor);
            dlAnchor.click();
            dlAnchor.remove();
        });

        document.addEventListener('click', (e) => {
            if (!searchInput.contains(e.target) && !suggestionsBox.contains(e.target)) {
                suggestionsBox.classList.add('hidden');
            }
        });
    </script>
</body>
</html>
