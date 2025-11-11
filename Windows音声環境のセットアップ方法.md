# Windows で静音ループバック文字起こしが安定した完全手順
このメモは `.env`、`Windows 音声環境の設定が上手く行った時の設定のスクショ` 配下のスクリーンショット & メモ、`easy_setup.cmd` の実行ログを突き合わせながら作成した「再現性の高い手順書」です。VB-Audio Virtual Cable を使って PC の再生音だけを拾い、イヤホンで静かにモニターしつつ Speechmatics で高精度に文字起こし・翻訳するところまでをまとめています。

---

## 0. 参照した資料
- `.env`: Speechmatics の接続設定、Google 翻訳のサービスアカウント、`AUDIO_DEVICE_INDEX=24`、`AUDIO_WINDOWS_LOOPBACK_DEVICE=CABLE Output (VB-Audio Virtual Cable)`、すべてのサンプルレートを 16 kHz/モノラルに統一した値など。
- `Windows 音声環境の設定が上手く行った時の設定のスクショ/*.png`: 既定デバイス、VB-CABLE の詳細プロパティ、音量ミキサー、Chrome/Discord の入出力固定などのスクリーンショット。
- `Windows 音声環境の設定が上手く行った時の設定のスクショ/windows-loopback-notes.txt`: イヤホンを既定のまま残しつつ Chrome だけを VB-CABLE に流すルーティングと、16 kHz にそろえた理由のまとめ。
- `Windows-VBCable  Ubuntu-pavucontrol.txt`: Stereo Mix／VB-CABLE／VoiceMeeter／PulseAudio など複数アプローチの安定性比較。Windows では VB-CABLE＋「このデバイスを聴く」が第一選択という結論。
- `scripts/easy_start.ps1`（`easy_setup.cmd` 経由）の実行ログ: 環境診断、ループバック候補、Speechmatics 接続ログ、翻訳ログまで全て残っている。

---

## 1. 構成イメージ
```
[Chrome/Discord など再生アプリ] --(出力=CABLE Input)--> [VB-CABLE 仮想ケーブル]
                                                            |
                                                            +--> [CABLE Output を Listen でヘッドホンへ]
                                                            |
                                                            +--> [Transcriber (AUDIO_DEVICE_INDEX=24) → Speechmatics → Google 翻訳]
```
既定の再生デバイスは Realtek ヘッドホンのままにしておき、ループバックしたいアプリだけを音量ミキサーで VB-CABLE に固定すると、他アプリや OS 効果音は巻き込まずに済みます。

---

## 2. PowerShell と仮想環境の準備
1. **Execution Policy**
   CurrentUser を RemoteSigned、Process を一時的に Bypass にして `Activate.ps1` やセットアップスクリプトがブロックされないようにします。
   ```powershell
   Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
   Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
   Get-ExecutionPolicy -List
   ```
   出力例:
   ```
   Process       Bypass
   CurrentUser   RemoteSigned
   LocalMachine  Unrestricted
   ```

2. **仮想環境**
   `.venv311\Scripts\Activate.ps1` を実行して以降の作業は全て同じ仮想環境で行います。
   ```powershell
   .\.venv311\Scripts\Activate.ps1
   ```

3. **デバイス一覧**
   `python -m transcriber.cli --list-devices` で VB-CABLE (WASAPI #24) を確認します。`ctranslate2` から `pkg_resources is deprecated` という警告が出ますが無視して問題ありません。
   ```
     22: IN      ステレオ ミキサー (2- Realtek(R) Audio)  (2)
     23: IN      マイク (2.0 Camera)                       (2)
   * 24: IN      CABLE Output (VB-Audio Virtual Cable)      (2)
   ```
   ここで得た番号を `.env` の `AUDIO_DEVICE_INDEX=24` に固定しておくと、Bluetooth 機器を抜き差ししてもパイプラインの入力先がブレません。

---

## 3. easy_setup.cmd / scripts\easy_start.ps1 での診断
`easy_setup.cmd` → `scripts\easy_start.ps1` の実行で以下を自動確認できます。

| 項目 | 具体的なログ | 意味 |
|------|--------------|------|
| Environment readiness | OS: Windows-11-10.0.26100、Python: 3.12.10、Executable: `.venv311\Scripts\python.exe` | 期待する実行環境かどうかを確認 |
| requirements preview | aiohttp〜websockets | requirements.txt の想定パッケージ |
| Package status | `[OK] aiohttp` など 10 件 | 依存関係が全て満たされている |
| Required files | `transcriber/*.py` の存在チェック | リポジトリ汚損の早期検知 |
| Logs & .env | `.env` が見つかり、`SPEECHMATICS_API_KEY`、`AUDIO_DEVICE_INDEX=24` などが設定済み | キーやデバイス設定漏れがないか確認 |
| Google credentials | `gen-lang-client-0219123936-d6e117f5a590.json` | 翻訳用サービスアカウント |
| OS guidance | `scripts/check_environment.py` や `setup_audio_loopback_windows.ps1` の案内 | 次に行うべき操作 |
| Audio diagnostics | ループバック候補 (#3/#12/#24/#37) と「設定済みデバイス: #24 CABLE Output」 | `.env` と Windows の設定が一致しているか確認 |

`Run setup_audio_loopback_windows.ps1 to configure loopback audio? [y/N]: y` と答えると、VB-CABLE の PnP 名称一覧が表示され、`mmsys.cpl` で探す際の迷子を防げます。

---

## 4. Windows 側のルーティング
### 4.1 役割分担
| 役割 | 設定内容 | スクリーンショット |
|------|----------|---------------------|
| 既定の再生デバイス | `ヘッドホン (2- Realtek(R) Audio)` を既定に保ち、耳からだけ音を聞く | `Windows 音声環境の設定が上手く行った時の設定のスクショ/sound-output-default-headphones.png` |
| 既定の録音デバイス | `マイク (2.0 Camera)` を既定に戻し、会議用マイクを殺さない | `Windows 音声環境の設定が上手く行った時の設定のスクショ/sound-input-default-camera-mic.png` |
| 仮想再生 (送信) | 対象アプリ（Chrome/Discord）の出力を `CABLE Input` に固定 | `Windows 音声環境の設定が上手く行った時の設定のスクショ/volume-mixer-chrome-loopback.png` 他 |
| 仮想録音 (受信) | `CABLE Output` を録音タブで選び「このデバイスを聴く」をオンにしてヘッドホンへモニター | `Windows 音声環境の設定が上手く行った時の設定のスクショ/recording-tab-cable-output-listen.png` |

### 4.2 具体手順
1. `Win + R` → `mmsys.cpl` で旧コントロールパネルを開く。
2. **再生タブ**: `CABLE Input` は「準備完了」のまま、`ヘッドホン` を既定に設定。
3. **録音タブ**: `CABLE Output` → プロパティ → 「このデバイスを聴く」をオン。再生デバイスにヘッドホンを指定し、ミュートせず音量 100%。
4. 録音タブのメーターで常時信号を確認。無音なら Chrome 側の出力デバイスがズレていないかチェック。
5. Windows 11 の「設定 > システム > サウンド」（`Windows 音声環境の設定が上手く行った時の設定のスクショ/sound-settings-entry-point.png`）でも既定デバイスが意図通りか再確認。

### 4.3 アプリ単位のルーティング
「設定 > サウンド > 音量ミキサー」で Chrome / Discord を以下に固定します。

- 出力デバイス: `CABLE Input (VB-Audio Virtual Cable)`
- 入力デバイス: `CABLE Output (VB-Audio Virtual Cable)`

これにより、YouTube や Discord の音は VB-CABLE に流れ、他アプリや Windows サウンドは Realtek ヘッドホン経由で静かに聴けます。

### 4.4 サンプルレートの統一
`windows-loopback-notes.txt` にもある通り、すべて 16 kHz / 16-bit / Mono に揃えるとドライバ側の自動リサンプリングが発生せず、レベルの揺れや遅延が減りました。

- `.env`: `AUDIO_DEVICE_SAMPLE_RATE=16000`、`AUDIO_SAMPLE_RATE=16000`、`AUDIO_CHANNELS=1`
- Windows: `CABLE Input` / `CABLE Output` の詳細設定を「16-bit, 16000 Hz」に設定（`cable-input-properties-16khz.png`, `cable-output-properties-16khz.png`）
- VB-CABLE の音量スライダーは 100%、Realtek は 70% 前後に調整

---

## 5. `.env` とパイプライン設定
```ini
TRANSCRIPTION_BACKEND=speechmatics
SPEECHMATICS_APP_ID=realtime
SPEECHMATICS_LANGUAGE=eo
SPEECHMATICS_CONNECTION_URL=wss://eu2.rt.speechmatics.com/v2
SPEECHMATICS_SAMPLE_RATE=16000

AUDIO_DEVICE_INDEX=24
AUDIO_DEVICE_SAMPLE_RATE=16000
AUDIO_SAMPLE_RATE=16000
AUDIO_CHANNELS=1
AUDIO_CHUNK_DURATION_SECONDS=0.5
AUDIO_WINDOWS_LOOPBACK_DEVICE=CABLE Output (VB-Audio Virtual Cable)

ZOOM_CC_ENABLED=false
TRANSCRIPT_LOG_ENABLED=true
TRANSCRIPT_LOG_PATH=logs/meet-session.log

WEB_UI_ENABLED=true
WEB_UI_OPEN_BROWSER=true

TRANSLATION_ENABLED=true
TRANSLATION_SOURCE_LANGUAGE=eo
TRANSLATION_TARGETS=ja,ko
TRANSLATION_PROVIDER=google
TRANSLATION_TIMEOUT_SECONDS=8.0
GOOGLE_TRANSLATE_CREDENTIALS_PATH=gen-lang-client-0219123936-d6e117f5a590.json
GOOGLE_TRANSLATE_MODEL=nmt
```

| キー | 役割 | 補足 |
|------|------|------|
| `TRANSCRIPTION_BACKEND` | Speechmatics を明示 | Whisper/Vosk に切り替える場合はここを変更 |
| `SPEECHMATICS_*` | API 接続情報 | 言語 `eo` (エスペラント)、サンプルレート 16 kHz を揃える |
| `AUDIO_*` | 入出力デバイス・チャンク設定 | `AUDIO_DEVICE_INDEX=24` がブレない限り WASAPI で安定 |
| `ZOOM_CC_ENABLED` | Meet では不要なため false | 誤送信防止 |
| `TRANSCRIPT_LOG_*` | `logs/meet-session.log` に追記 | easy_start 実行ログと一致 |
| `WEB_UI_*` | 自動で http://127.0.0.1:8765 を開く | ブラウザで字幕確認 |
| `TRANSLATION_*` | Speechmatics → Google 翻訳（ja, ko） | サービスアカウント JSON をパス指定 |

---

## 6. 運用フロー（再起動〜文字起こし開始まで）
1. 物理接続を確認（ヘッドホンは Realtek、マイクはカメラ）。
2. PowerShell で Execution Policy を設定し `.venv311` を有効化。
3. `python -m transcriber.cli --list-devices` で #24 を再確認（番号が変わっていたら `.env` を更新）。
4. `easy_setup.cmd` を実行し、環境チェックが全て `[OK]` になることを確認。
5. `setup_audio_loopback_windows.ps1` の質問に `y` で答えて VB-CABLE デバイスを再確認。
6. `Environment ready. Start transcription now? [Y/n]: Y` でパイプラインを開始。
7. ログに以下の流れが出れば成功。
   ```
   [INFO] root: Starting transcription pipeline with backend=speechmatics.
   [INFO] root: Starting audio stream on CABLE Output ... (device index 24)
   [INFO] root: Audio stream started successfully
   [INFO] root: Connected to Speechmatics realtime endpoint.
   [INFO] root: Final: Saluton spektantoj.
   ```
8. http://127.0.0.1:8765 に自動でブラウザが開き、ログは `logs/meet-session.log` に追記されます。

---

## 7. モニタリングと検証方法
- **`python -m transcriber.cli --diagnose-audio`**: ループバック候補の一覧と設定済みデバイス (#24) を毎回確認できる。
- **録音タブのレベルメーター**: `CABLE Output` のメーターが揺れていればルートは生きている。無音なら Chrome の出力や音量ミキサーを再確認。
- **「このデバイスを聴く」**: イヤホンからループバック音を直接モニターできるため、録れているか耳でも判断できる。
- **`logs/meet-session.log`**: 22:40 台に「Final: Saluton spektantoj.」「Final: Hodiaŭ mi volas montri al vi Esperanto kafejon en Tokio.」などが記録され、翻訳文も併記される。
- **Web UI**: ブラウザで字幕を見ながらリアルタイムに確認。`aiohttp` のアクセスログも残る。

---

## 8. マイク入力を殺さないための運用
1. 既定の録音デバイスは常に `マイク (2.0 Camera)`。VB-CABLE を既定にしない。
2. Chrome/Discord の **入力デバイス** は用途に応じて切り替える。例: ChatGPT 音声入力を使うときは入力=実マイク、出力=CABLE Input。
3. `chrome://settings/content/microphone` で使用マイクを固定し、サイトごとの許可を確認。
4. どうしても実マイクとシステム音を同時に配信したい場合は VoiceMeeter へ切り替える、ただし本構成では相手音のみを取ることに集中する。

---

## 9. トラブルシューティング
| 症状 | よくある原因 | 対処 |
|------|--------------|------|
| `sounddevice.PortAudioError` でデバイスを開けない | BT ヘッドセットなどで番号が変わった | `--list-devices` を再実行し `.env` の `AUDIO_DEVICE_INDEX` を更新 |
| ループバックに音が乗らない | Chrome/Discord の出力先が既定のまま、または「このデバイスを聴く」がオフ | 音量ミキサーで再設定し、録音タブで Listen を確認 |
| 音量が小さい/揺れる | 48 kHz ↔ 16 kHz の再サンプルが発生 | VB-CABLE と `.env` のサンプルレートを 16 kHz に統一し、音量を 100% 付近に固定 |
| 翻訳が返ってこない | Google 認証 JSON のパス違い | `.env` の `GOOGLE_TRANSLATE_CREDENTIALS_PATH` とファイルの実体を確認 |
| `pkg_resources is deprecated` 警告 | ctranslate2 が setuptools を参照 | Setuptools < 81 を pin するか、無視して継続（実際の処理には影響なし） |

---

## 10. 他手法との比較（`Windows-VBCable  Ubuntu-pavucontrol.txt` から抜粋）
- **VB-CABLE + 「このデバイスを聴く」**: 追加ハード不要で機種依存が少なく、イヤホンを既定のまま保てる。今回採用。
- **Stereo Mix**: 動けば低遅延だが、Realtek ドライバ依存が強く、イヤホンを挿すと無音になる個体があるため再現性が低い。
- **VoiceMeeter Banana**: マイクとシステム音を同時に扱いやすいが、設定項目が多く初心者には複雑。マイクも合成したい場合の次善策。
- **PulseAudio Monitor / PipeWire**: Ubuntu 向けの手法。Windows では追加ソフト不要な VB-CABLE が最優先という結論。

---

## 11. スクリーンショット索引
| ファイル | 内容 |
|---------|------|
| `all-sound-devices-list.png` | サウンド設定の全デバイス一覧 |
| `cable-input-properties-16khz.png` | `CABLE Input` の詳細設定 (16-bit/16 kHz) |
| `cable-output-properties-16khz.png` | `CABLE Output` の詳細設定 |
| `playback-devices-classic-panel.png` | `mmsys.cpl` の再生タブ |
| `recording-tab-cable-output-listen.png` | 「このデバイスを聴く」のオン設定 |
| `sound-settings-entry-point.png` | Windows 11 の設定画面入口 |
| `volume-mixer-chrome-loopback.png` / `volume-mixer-discord-loopback.png` | アプリ別ルーティング設定 |
| `sound-output-default-headphones.png` / `sound-input-default-camera-mic.png` | 既定デバイスの状態 |

---

## 12. 最終チェックリスト
1. `.venv311` が有効で `python -m transcriber.cli --list-devices` に #24 が見えている。
2. `.env` の `AUDIO_DEVICE_INDEX` と Windows の `CABLE Output` 名称が一致。
3. `CABLE Output` に「このデバイスを聴く」が設定され、イヤホンからモニターできる。
4. Chrome/Discord の出力先が `CABLE Input`、入力先が `CABLE Output`。
5. easy_setup の診断で全項目 `[OK]`。`logs/meet-session.log` に Speechmatics の `Final:` 行が記録される。

以上を満たせば、イヤホンで静かに聞きながら PC 出力だけを確実に回収し、Speechmatics + Google 翻訳で高精度な字幕運用が安定して再現できます。
