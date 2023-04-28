# bp-hands-on-2023
NICAR23

## 概要
- このリポジトリは2023年3月、米国ナッシュビルにて開催されたデータジャーナリズムカンファレンスのNICARで受講したハンズオン講座をWindows向けに改変したものです。

- 元のリポジトリはAndrew Milligan氏のものです。

- 講座のdescription(https://schedules.ire.org/nicar-2023#2285)
>This class will be a hands-on walkthrough of the methods used by Milligan to make the map files that power live, interactive election results maps, inspired heavily by Mike Bostock's Command-Line Cartography Series. (bit.ly/cli-carto)
>
>We'll dig into some popular JSON-based map formats, ways to convert between them, and, ultimately, how to bend them to your will. By the end we'll be able to take a Census shapefile, extract from it exactly what we need, project it, and turn it into either a static image or a TopoJSON ready for your interactive web app. And we'll do it all from the command line using open-source tools, so you can wrap it up in a script and run it again and again!
>
>This session would be good for people with some command-line experience who are comfortable getting knee-deep in some JSON.

## Making maps with command-line tools for Windows11
### 使用するソフトウェア
- Powershell
- Ubuntu(WSL※Windows Subsystem for Linux)
- VScode
### 手順
1. Powershellを管理者権限で開く
2. Ubuntu(wsl)をインストール
```
wsl --install
```
3. Ubuntuを開く
4. UbuntuにIDとPWを設定。PC再起動
5. Powershellにてインストールできているか確認
```
wsl -l -v
```
6. VScodeのインストールhttps://code.visualstudio.com/
7. VScodeを起動
8. Open folderからダウンロードしたリポジトリを開く
9. リポジトリを開いた状態で、Ctrl+` でターミナルを開く
10. VScodeのTerminalをUbuntuに変更。Linuxコマンドが使用可能になる
11. Ubuntuのターミナルを開く
12. セットアップ１
```
bin/setup_class.sh
```
13. セットアップ２
```
source bin/configure_shell.sh
```
14. ディレクトリを３つ作成する
```
mkdir work outputs inputs
```
15. 地点別データをinputsにコピーする
```
cp data/tn_population_by_tract.csv inputs/
```
16. 地図データのZipをinputsにコピーする
```
cp shapefiles/cb_2021_47_tract_500k.zip inputs/
```
17. 地図データのZipファイルを解凍する
```
unzip -o -d inputs/ inputs/cb_2021_47_tract_500k.zip
```
18. パスを通す
```
export PATH=$PATH:./node_modules/.bin
```
19. シェープファイルをgeojsonに変換
```
shp2json inputs/cb_2021_47_tract_500k.shp work/tn_tracts.geo.json
```
20. 平面の地図にする
```
cat work/tn_tracts.geo.json \
  | geoproject 'd3.geoConicEqualArea().parallels([35.4,36.3]).rotate([86, 0]).fitSize([960, 960], d)' \
  > work/tn_tracts_projected.geo.json
```
21. geojsonをndjson形式に分割
```
ndjson-cat work/tn_tracts_projected.geo.json \
  | ndjson-split 'd.features' \
  > work/tn_tracts_projected.ndjson
```

22. csvデータをndjsonに変換する
```
csv2json -n inputs/tn_population_by_tract.csv \
  | ndjson-map '{ id: d.geoid.split("US")[1], population: +d.B01003001 }' \
  > work/tn_population_by_tract.ndjson
```
23. 地図とcsvデータを結合する
```
ndjson-join 'd.id' 'd.properties.GEOID' \
  work/tn_population_by_tract.ndjson \
  work/tn_tracts_projected.ndjson \
  > work/tn_population_and_tracts_projected.ndjson
```
24. 結合したndjsonから人口密度を計算
```
cat work/tn_population_and_tracts_projected.ndjson \
  | ndjson-map '
      population = d[0].population,
      m2permi2 = 1609.34 * 1609.34,
      area = d[1].properties.ALAND,
      d[1].properties.density = Math.floor(population / area * m2permi2),
      d[1]
    ' \
  > work/tn_tracts_projected_with_population.ndjson
```
25. 新しいndjsonからgeojsonを作成
```
cat work/tn_tracts_projected_with_population.ndjson \
  | ndjson-reduce \
  | ndjson-map '{ type: "FeatureCollection", features: d }' \
  > work/tn_tracts_projected_with_population.geo.json
```
26. 地図をgeojson形式からtopojson形式に変換
```
cat work/tn_tracts_projected_with_population.geo.json \
  | geo2topo tracts=- \
  > work/tn_tracts_projected_with_population.topo.json
```
27. topojsonをきれいに整形する
```
cat work/tn_tracts_projected_with_population.topo.json \
  | toposimplify -p 1 -f \
  > work/tn_tracts_projected_simplified_with_population.topo.json
```
28. さらに整形。ファイルサイズを小さくする
```
cat work/tn_tracts_projected_simplified_with_population.topo.json \
  | topoquantize 1e5 \
  > work/tn_tracts_projected_simplified_quantized_with_population.topo.json
```
29. 色を追加する
```
cat work/tn_tracts_projected_simplified_quantized_with_population.topo.json \
  | topo2geo tracts=- \
  | ndjson-map \
      -r d3array=d3-array \
      -r d3scale=d3-scale \
      -r d3chroma=d3-scale-chromatic \
      '
        z = d3scale.scaleSequential(d3chroma.interpolateMagma).domain([100, 0]),
        d.features.forEach((f) => {
          f.properties.fill = z(Math.sqrt(f.properties.density));
        }),
        d
      ' \
  | ndjson-split 'd.features' \
  > work/tn_tracts_projected_simplified_quantized_with_population_colors.ndjson
```
30. 色をSVGに追加する
```
cat work/tn_tracts_projected_simplified_quantized_with_population_colors.ndjson \
  | geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  > outputs/tn_tract_population_density_sqrt.svg
```
31. 郡境を追加する
```
cat work/tn_tracts_projected_simplified_quantized_with_population.topo.json \
  | topomerge -k 'd.properties.GEOID.slice(0, 5)' counties=tracts \
  | topomerge --mesh -f 'a !== b' counties=counties \
  | topo2geo -n counties=- \
  | ndjson-map '
      d.properties = {"stroke": "#000", "stroke-opacity": 0.3},
      d
    ' \
  > work/tn_counties.ndjson
```
32. 郡境をSVGに追加する

```
(
  cat work/tn_tracts_projected_simplified_quantized_with_population_colors.ndjson;
  cat work/tn_counties.ndjson;
) \
  | geo2svg -n --stroke none -p 1 -w 960 -h 960 \
  > outputs/tn_tract_population_density_sqrt.svg
```

