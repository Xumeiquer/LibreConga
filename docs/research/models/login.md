# Login

## 5490

### Request

```protobuf
message LoginRequest {
    required string email = 1;
    required string password = 2;
    required int32 unknown = 4; // always set as 1003d
}
```

### Response

```protobuf
// Login failed
message LoginResponse {
    required int32 unknown = 1;
    required string error = 2; // User password is incorrect(username: <email>)
}
```

```protobuf
// Login success
message LoginResponse {
    message Unknown {
        required int32 unknown1 = 1;
        required int32 unknown2 = 2;
    }

    message Msg {
        required int32 user_id = 1;
        required string session_id = 2;
        required int32 unknown = 3;
        required int32 unknown1 = 5;
        required Unknown unknown2 = 6;
    }

    required int32 unknown = 1;
    required Msg msg = 3;
}
```
