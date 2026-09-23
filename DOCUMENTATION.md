# Aditus - Setup and Technical Reference

This document contains the setup instructions and implementation details that accompany the [project walkthrough](README.md). It describes the code in this repository. The [original report](report.pdf) provides design context, but some flows and limitations differ from the implementation here.

## Contents

- [Requirements](#requirements)
- [Backend setup](#backend-setup)
- [Client configuration](#client-configuration)
- [Phone app](#phone-app)
- [ESP32 controller](#esp32-controller)
- [Watch app](#watch-app)
- [BLE protocol](#ble-protocol)
- [Access rules](#access-rules)
- [Backend API](#backend-api)
- [Storage and code layout](#storage-and-code-layout)
- [Known limitations](#known-limitations)

## Requirements

| Component | Requirements |
| --- | --- |
| Backend | Python 3, `pip`, and `venv`; dependencies are pinned in [requirements.txt](aditus_backend_service/requirements.txt) |
| Flutter apps | The committed lockfiles require Flutter 3.35.0 or later and Dart `>=3.9.2 <4.0.0`; an Android development toolchain is also needed |
| Phone | An Android device with BLE for the hardware flow |
| Watch | A Wear OS device with BLE; the Android project sets `minSdk = 26` |
| Controller | An ESP32 board with Wi-Fi and BLE, a USB cable, and an Arduino development environment with ESP32 board support |
| Network | Backend access from the phone and watch during setup, and from the ESP32 during every unlock |

The repository includes Android platform projects for both apps. Use physical devices to exercise the BLE flow with the controller. The Arduino core and library versions are not pinned.

## Backend Setup

From the repository root:

```bash
cd aditus_backend_service
bash setup.sh
```

The script creates `venv`, installs the requirements, and copies `.env.example` to `.env` if that file does not already exist. Edit `.env` before starting the service:

| Variable | Purpose |
| --- | --- |
| `FLASK_ENV` | Use `development` for this local setup |
| `FLASK_HOST`, `FLASK_PORT` | Development server bind address and port; defaults are `0.0.0.0` and `5000` |
| `SECRET_KEY`, `JWT_SECRET_KEY` | Set your own application and JWT secrets |
| `DATABASE_URL` | Defaults to `sqlite:///aditus.db` |
| `CORS_ORIGINS` | Comma-separated origins; the template allows `*` |
| `ESP32_API_KEY` | Shared controller API key; must match the firmware |
| `ADMIN_EMAIL`, `ADMIN_PASSWORD` | Credentials for the initial administrator |
| `ADMIN_FIRST_NAME`, `ADMIN_LAST_NAME` | Name used when creating that administrator |

Then start the service:

```bash
source venv/bin/activate
python run.py
```

The setup script does not initialize the database. [`create_app()`](aditus_backend_service/app/__init__.py) creates tables and the initial administrator when the application starts. The administrator is created only if no admin account exists; changing the environment variables later does not reset an existing account. With the default SQLite URI, the database is stored in Flask's `instance` directory.

Check the local server from another terminal:

```bash
curl http://127.0.0.1:5000/health
```

Expected JSON:

```json
{"status": "healthy", "service": "Aditus Backend"}
```

## Client Configuration

The clients contain the original deployment address as a source constant. Set each of these locations to your backend:

| File | Setting | URL shape |
| --- | --- | --- |
| [Phone API constants](smartphone_client_app/lib/core/constants/api_constants.dart) | `ApiConstants.baseUrl` | `https://YOUR_BACKEND/api` |
| [Phone history BLoC](smartphone_client_app/lib/features/history/presentation/bloc/history_bloc.dart) | `HistoryBloc.baseUrl` | `https://YOUR_BACKEND/api` |
| [Watch pairing service](smartwatch_client_app/lib/services/pairing_service.dart) | `PairingService.baseUrl` | `https://YOUR_BACKEND/api` |
| [ESP32 firmware](esp32_door_controller/esp32_door_controller.ino) | `API_BASE_URL` | `https://YOUR_BACKEND/` |

The history screen has its own URL constant, so updating only `ApiConstants` is not enough. The firmware base URL needs its trailing slash because request paths are appended directly.

The Flask development server serves HTTP. The firmware uses `WiFiClientSecure`, so a complete setup with that client requires an HTTPS endpoint in front of Flask. Physical devices need a network address they can reach; `127.0.0.1` on a phone or watch refers to that device. The original hosted address is not required to run your own instance.

## Phone App

From the repository root, with an Android device connected:

```bash
cd smartphone_client_app
flutter pub get
flutter devices
flutter run -d PHONE_DEVICE_ID
```

Replace `PHONE_DEVICE_ID` with the identifier shown by `flutter devices`.

1. Sign in with the administrator configured in the backend, or an account created by that administrator.
2. Choose a four-to-six-digit PIN. Optionally enable biometric authentication if available.
3. Name and register the phone. This generates its RSA key pair and sends the public key to the backend.
4. Enable Bluetooth and grant the permissions requested for discovery. Both BLE services explicitly request location permission.
5. Use **Management** to create users, groups, and doors. Use **Account** to view devices, change security settings, or pair a watch.

On later startup, `AuthBloc` checks the stored session against the backend, attempts a token refresh on an authentication failure, and routes the user through the local PIN or biometric screen. The default backend token lifetimes are three days for access tokens and thirty days for refresh tokens.

## ESP32 Controller

Open [`esp32_door_controller.ino`](esp32_door_controller/esp32_door_controller.ino) in an Arduino environment with ESP32 support. Install ArduinoJson; the sketch also uses the ESP32 core's Wi-Fi, HTTP, BLE, and mbedTLS headers. Select the board and serial port that match your hardware.

Set these constants before flashing:

| Constant | Value to supply |
| --- | --- |
| `WIFI_SSID` | Your Wi-Fi network name |
| `WIFI_PASSWD` | Your Wi-Fi password |
| `ESP32_API_KEY` | The same key as the backend environment variable |
| `API_BASE_URL` | Your reachable HTTPS backend root, including the trailing slash |

To register the controller:

1. Flash the sketch and open the serial monitor at **115200 baud**.
2. Copy the **BLE door MAC** printed during initialization. This is the BLE address, not the Wi-Fi address.
3. In the phone app, open **Management → Door Management → Create Door**. Enter the door name and the BLE address in the device identifier field, using uppercase colon-separated notation. Keep the door active.
4. The ESP32 retries configuration every ten seconds until the backend recognizes its address. It then retrieves the door ID and name and starts BLE advertising.
5. Open the door's **Manage Access Control** screen and grant your user or a group access. Admin status alone does not grant door access.
6. Scan from the phone and request an unlock. A valid request produces an `AUTHORIZED` notification and a three-second LED indication.

| LED behavior | Meaning |
| --- | --- |
| Steady on | Controller is registered and idle |
| Slow blink | Waiting for backend door configuration after an unsuccessful registration |
| Fast blink for three seconds | Successful unlock indication |

[`unlockDoor()`](esp32_door_controller/esp32_door_controller.ino) only changes the LED state. The repository does not include relay wiring or physical lock actuation. The firmware's logging URL also needs the correction described under [known limitations](#known-limitations).

## Watch App

From the repository root, with the Wear OS device connected:

```bash
cd smartwatch_client_app
flutter pub get
flutter devices
flutter run -d WATCH_DEVICE_ID
```

Replace `WATCH_DEVICE_ID` with the identifier shown by `flutter devices`.

1. On the phone, open **Account → Pair Smartwatch**.
2. On the watch, select **Start Pairing** and enter the six-digit code shown by the phone.
3. The watch generates its own RSA key pair and sends its public key, device name, and code to the backend. The session expires after five minutes and can be used once.
4. The backend creates the watch's device record and returns it. The watch stores the device ID and owner ID; this flow does not issue JWTs to the watch.
5. After pairing, the watch scans for doors and offers an unlock button for the strongest signal. It uses the same BLE challenge-response protocol as the phone.

The watch needs backend connectivity for pairing. Subsequent unlock requests use BLE, with the ESP32 making the backend requests. **Reset Smartwatch** clears the watch's local storage. Delete the watch through **My Devices** on the phone to remove its backend registration.

## BLE Protocol

Both clients filter advertisements by service UUID `4fafc201-1fb5-459e-8fcc-c5c9c331914b`.

| Characteristic | UUID | Properties | Payload |
| --- | --- | --- | --- |
| Identity | `5f2e6f9a-6f3d-4a1b-8f0a-7b8b5a3d0b1a` | Write | UTF-8 JSON containing `user_id` and `device_id` |
| Challenge | `6a3d9e2c-2a9a-4c1b-8f0a-7b8b5a3d0b1a` | Read, notify | Six-digit challenge as a string |
| Signature | `7b8b5a3d-0b1a-4c1b-8f0a-6f3d9e2c2a9a` | Write | Base64-encoded RSA signature |
| Status | `8f0a7b8b-5a3d-4c1b-8f0a-6f3d9e2c2a9a` | Read, notify | `AUTHORIZED` or a `DENIED_...` status |

An example identity payload is:

```json
{"user_id": "1", "device_id": "3"}
```

Here `device_id` means the registered phone or watch's database ID. It is different from the door model's `device_id`, which stores the ESP32's BLE MAC address.

The clients set up challenge and status notifications before writing the identity payload. The controller checks the user's permission and fetches the device public key, then sends a challenge. The client signs the UTF-8 challenge bytes using PointyCastle's `SHA-256/RSA` signer. It sends the Base64 result with `allowLongWrite: true` because the encoded RSA-2048 signature is 344 characters long.

The ESP32 fetches the public key again when the signature arrives. It decodes the signature, parses the PEM public key, hashes the challenge with SHA-256, and verifies it using mbedTLS. Both client wait stages have thirty-second timeouts; the controller also has a thirty-second signature timeout. Disconnecting resets the controller's unlock state.

Status examples include `DENIED_no_permission`, `DENIED_door_inactive`, `DENIED_KEY_FETCH_FAILED`, `DENIED_INVALID_SIGNATURE`, and `DENIED_TIMEOUT`. The reason suffix is case-sensitive and is mapped to display text by each client's unlock BLoC.

## Access Rules

The controller calls `POST /api/doors/check-access`. That route checks that the user and door exist and that the door is active, then calls [`User.has_access_to_door()`](aditus_backend_service/app/models/user.py).

The actual evaluation order is:

1. If the user is in the door's user exceptions, deny access.
2. If the user has a direct door grant, allow access.
3. For each group the user belongs to, allow access if that group has a grant and is not in that door's group exceptions.
4. Otherwise deny access.

A group exception excludes that group's grant. It does not override a direct user grant or a grant through another group. A user exception does override those grants. The older report and backend README describe a broader deny-first rule for groups; the code implements the narrower behavior above.

The backend returns `direct_access` or `group_access` on success. Permission failures use `no_permission`; this route does not return separate user-exception or group-exception reasons. RSSI is used by the clients for ordering doors, not as a backend distance check.

## Backend API

Phone API calls use `Authorization: Bearer <access_token>`. The refresh endpoint requires the refresh token instead. Controller endpoints require `api_key` in the JSON request body. Administration routes also check the account's role.

The following are the main integration routes. Trailing slashes shown here match the Flask declarations.

| Method | Route | Caller / purpose |
| --- | --- | --- |
| `GET` | `/health` | Public health check |
| `POST` | `/api/auth/login` | Email/password login; returns tokens and user |
| `POST` | `/api/auth/refresh` | Refresh-token authentication; returns an access token |
| `GET` | `/api/users/me` | Current user and associated account data |
| `PUT` | `/api/users/me/password` | Change password with current and new passwords |
| `POST` | `/api/devices/` | Register a phone with `name` and `public_key` |
| `GET` | `/api/devices/my-devices` | List the current user's devices |
| `DELETE` | `/api/devices/<device_id>` | Owner or administrator revokes a device |
| `POST` | `/api/devices/pairing/initiate` | Authenticated phone requests a pairing code |
| `POST` | `/api/devices/pairing/complete` | Watch submits `code`, `device_name`, and `public_key` |
| `POST` | `/api/doors/configure` | Controller submits `mac_address` to retrieve door ID and name |
| `POST` | `/api/doors/check-access` | Controller submits `user_id` and `door_id` for a permission decision |
| `POST` | `/api/devices/<device_id>/public-key` | Controller fetches a registered device's public key |
| `POST` | `/api/access-logs/` | Controller submits an access result |
| `GET` | `/api/access-logs/my-logs` | Current user's paginated history |
| `GET` | `/api/access-logs/` | Administrator reads and filters system logs |

User, group, and door creation, updates, and deletion are implemented in the [route modules](aditus_backend_service/app/routes/). Door access rules live under `/api/doors/<door_id>/access`, with separate `users`, `groups`, `exceptions/users`, and `exceptions/groups` routes for adding and removing entries. Group membership changes live under `/api/groups/<group_id>/members`.

Access records contain the user, door, optional device, action, success flag, optional failure reason, timestamp, and optional device/IP metadata. Log queries use `limit` and `offset`. The system log route additionally accepts `user_id`, `door_id`, `device_id`, `success`, `from`, and `to`. The phone history screen requests twenty records per page.

## Storage and Code Layout

The phone uses `flutter_secure_storage` with encrypted shared preferences for JWTs, PEM keys, the registered device ID, cached user data, and the PIN hash and salt. The PIN service uses salted SHA-256. The watch uses the same storage package for its keys and device/owner IDs. Keys are generated in Dart and stored as PEM strings; the application does not generate non-exportable RSA keys in Android hardware.

The backend stores users, devices, doors, groups, access logs, and pairing sessions through SQLAlchemy. Five association tables represent group membership, user and group grants, and user and group exceptions. SQLite is the default database; no database migration system is included.

```text
aditus_backend_service/
  app/models/              # Persistent entities and permission evaluation
  app/routes/              # JSON API blueprints
  app/utils/               # Role/API-key checks and initial admin creation
  app/config.py            # Environment and JWT configuration
  run.py                   # Development server entry point
esp32_door_controller/
  esp32_door_controller.ino # BLE, Wi-Fi, verification, and LED states
smartphone_client_app/lib/
  core/                    # APIs, BLE, security, navigation, theme, shared UI
  features/                # Auth, devices, groups, unlock, history, account, admin
smartwatch_client_app/lib/
  bloc/                    # Discovery and unlocking state
  screens/                 # Pairing, main view, and settings
  services/                # BLE, cryptography, pairing, and storage
screenshots/               # Phone/watch screenshots, demo GIF, and video
report.pdf                 # Original project report
```

## Known Limitations

- **Controller logging URL:** `logAccessAttempt()` posts to `api/logs/access`, but Flask registers `POST /api/access-logs/`. To receive controller logs, change that appended path in the firmware to `api/access-logs/`. Some paths, including the controller's signature timeout and invalid-state response, do not call the logging function at all. There is no persistent retry queue, so logging is not guaranteed for every attempt.
- **Unfinished navigation:** the phone's System Logs entry, user/door log shortcuts, and the group's Manage Door Access shortcut display placeholder messages. The personal history screen and backend log endpoints exist; group grants can be managed from a door's access screen.
- **Hardware and connectivity:** successful verification blinks an LED. Physical lock actuation and controller-side offline authorization are not implemented. The phone also requires a backend session check at startup.
- **Watch reliability:** the report describes intermittent ESP32 hangs during Pixel Watch unlocks. It suspects BLE behavior, but the repository does not establish the cause or include a confirmed fix.
- **Controller transport and credentials:** HTTPS calls use `setInsecure()`, which disables certificate verification. Wi-Fi credentials and a shared API key are constants in the sketch. Verified TLS and protected, per-controller credentials would be needed for a deployment.
- **Challenge and identity checks:** the challenge is generated with Arduino `random()` as a six-digit number. A stronger design needs cryptographic nonces and explicit binding between the requested user, registered device owner, door, and session. The current firmware uses the returned public key without checking its associated owner against the submitted user ID.
- **Sessions and pairing:** the logout route does not revoke issued JWTs, and the pairing-code completion route has no rate limiting. Resetting the watch only clears local storage and leaves the backend device record in place.
- **Verification:** no application test suite is included. The screenshots, video, and report record the original demonstration; they do not establish a fresh hardware validation of this repository snapshot.
