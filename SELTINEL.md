```gee
// Função para filtrar e obter a imagem Sentinel
function getSentinelImage(area, startDate, endDate) {
  // Filtra a coleção Sentinel para a área e período especificados
  var sentinel = ee.ImageCollection("COPERNICUS/S2_SR")
    .filterBounds(area)
    .filterDate(startDate, endDate)
    .filterMetadata('CLOUDY_PIXEL_PERCENTAGE', 'less_than', 10)
    .filterMetadata('CLOUD_COVERAGE_ASSESSMENT', 'less_than', 10);

  // Obtém o mosaico da mediana e ajusta a escala
  var image = ee.Image(sentinel.median()).clip(area).multiply(0.0001);

  // Retorna a imagem e a coleção
  return { image: image, collection: sentinel };
}

// Função para adicionar camada ao mapa
function addMapLayer(image, area, areaName) {
  // Adiciona a área ao mapa
  Map.addLayer(area, null, areaName);

  // Adiciona a imagem RGB Sentinel 2 ao mapa
  Map.addLayer(image, { bands: ['B4', 'B3', 'B2'], min: 0, max: 0.3 }, 'RGB Sentinel 2 - ' + areaName);

  // Calcula e adiciona a camada NDVI ao mapa
  var ndvi = image.normalizedDifference(['B8', 'B4']);
  Map.addLayer(ndvi, { min: 0.04, max: 1.0, palette: ['#ff2a00', '#ff8303', '#fdc120', '#b3c01a', '#4da910', '#1e7d83', '#1e7d83'] }, "NDVI Sentinel 2 - " + areaName, true);

  // Calcula e adiciona a camada SAVI ao mapa
  var savi = image.expression(
    '1.5*((NIR-RED)/(NIR+RED+0.5))', {
      'NIR': image.select('B8'),
      'RED': image.select('B4'),
    }).rename('savi');
  Map.addLayer(savi, { 'min': 0.0, 'max': 1, 'palette': ['#ff2a00', '#ff8303', '#fdc120', '#b3c01a', '#4da910', '#1e7d83', '#1e7d83'] }, "SAVI Sentinel 2 - " + areaName, true);

  // Centraliza o mapa na área de interesse
  Map.centerObject(area, 8);
}

// Função para exportar a camada NDVI para o Google Drive
function exportNDVIToDrive(ndvi, description, folder, area, image) {
   // Exporta a imagem para o Google Drive
  Export.image.toDrive({
    image: ndvi,
    folder: folder,
    description: description,
    region: area,
    scale: 10,
    maxPixels: 1e13,
    fileFormat: 'GeoTIFF',
    formatOptions: {
      cloudOptimized: true
    },
  });
}

// Lista de áreas de estudo
var areasEstudo = [
  // Substitua os paths pelas suas áreas de estudo
  { area: ee.FeatureCollection('projects/ee-matocompeticao2023/assets/BR_TALHOES_NDVI_SET_APO'), name: 'TALHOES_NDVI_SET_APO' },
  { area: ee.FeatureCollection('projects/ee-matocompeticao2023/assets/BR_TALHOES_NDVI_SET_CPO'), name: 'TALHOES_NDVI_SET_CPO' },
  { area: ee.FeatureCollection('projects/ee-matocompeticao2023/assets/BR_TALHOES_NDVI_SET_MGS'), name: 'TALHOES_NDVI_SET_MGS' },
  { area: ee.FeatureCollection('projects/ee-matocompeticao2023/assets/BR_TALHOES_NDVI_SET_SNG'), name: 'TALHOES_NDVI_SET_SNG' }
];

// Período de datas
var startDate = '2023-10-01';
var endDate = '2023-10-22';

// Iterar sobre as áreas de estudo
areasEstudo.forEach(function(areaObj) {
  // Obtém a imagem Sentinel para a área e período especificados
  var result = getSentinelImage(areaObj.area, startDate, endDate);
  var sentinelImage = result.image;
  var sentinelCollection = result.collection;

  // Imprime o intervalo de datas
  var range = sentinelCollection.reduceColumns(ee.Reducer.minMax(), ["system:time_start"]);
  print('Intervalo de datas para', areaObj.name + ':', ee.Date(range.get('min')), ee.Date(range.get('max')));

  // Calcula o NDVI
  var ndvi = sentinelImage.normalizedDifference(['B8', 'B4']);

  // Adiciona camadas ao mapa para visualização, incluindo o nome da área
  addMapLayer(sentinelImage, areaObj.area, areaObj.name);

  // Modifica a descrição conforme necessário e exporta o NDVI para o Google Drive
  var description = 'NDVI_' + areaObj.name;
  exportNDVIToDrive(ndvi, description, 'Indice de Vegetacao', areaObj.area, sentinelImage);
});
```
