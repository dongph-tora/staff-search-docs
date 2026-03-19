# [STAFF-02] Unique 6-Digit Staff ID Generation — Implementation Plan

**Spec:** `docs/screens/STAFF-02-staff-number-generation.md`
**Priority:** P1 MVP
**Phụ thuộc:** DB-01 (`staff_profiles.staff_number` column đã được define), STAFF-01 (StaffService.CreateProfile gọi service này)

> Backend-only ticket. Không có Flutter changes.

---

## Backend Steps

### Step 1: Thêm error sentinel

**File:** `internal/model/error.go` (thêm vào)

```go
var ErrStaffNumberExhausted = errors.New("staff_number_exhausted")
```

### Step 2: Thêm StaffNumberExists vào StaffRepository

**File:** `internal/repository/staff_repository.go` (thêm method)

Method `StaffNumberExists(ctx context.Context, number string) (bool, error)`:
```go
var count int64
err := r.db.WithContext(ctx).
  Model(&model.StaffProfile{}).
  Where("staff_number = ?", number).
  Count(&count).Error
if err != nil {
  return false, err
}
return count > 0, nil
```

### Step 3: Tạo StaffNumberService

**File:** `internal/service/staff_number_service.go` (tạo mới)

```go
package service

import (
  "context"
  "crypto/rand"    // PHẢI dùng crypto/rand, không dùng math/rand
  "fmt"
  "math/big"
  "staff-search-api/internal/model"
  "staff-search-api/internal/repository"
)

type StaffNumberService struct {
  staffRepo *repository.StaffRepository
}

func NewStaffNumberService(staffRepo *repository.StaffRepository) *StaffNumberService {
  return &StaffNumberService{staffRepo: staffRepo}
}
```

Method `Generate(ctx context.Context) (string, error)`:
```
maxAttempts = 10
for attempt = 1 to maxAttempts:
  1. Dùng crypto/rand để generate random int trong [0, 999999]:
     n, err := rand.Int(rand.Reader, big.NewInt(1000000))
     if err != nil { return "", err }
  2. Format zero-padded 6 digits: fmt.Sprintf("%06d", n.Int64())
  3. Gọi staffRepo.StaffNumberExists(ctx, number)
  4. Nếu err != nil → return "", err
  5. Nếu !exists → return number, nil
return "", model.ErrStaffNumberExhausted
```

### Step 4: Inject StaffNumberService vào StaffService

**File:** `internal/service/staff_service.go` (modify)

- Thêm `staffNumberService *StaffNumberService` vào struct
- Update constructor để nhận và inject `staffNumberService`
- Trong `CreateProfile()`: gọi `staffNumberService.Generate(ctx)` để lấy `staffNumber` trước khi insert

### Step 5: Wire dependencies trong main.go

**File:** `main.go`

```go
staffRepo := repository.NewStaffRepository(db)
staffNumberService := service.NewStaffNumberService(staffRepo)
staffService := service.NewStaffService(staffRepo, staffNumberService, userRepo)
```

---

## Flutter Steps

Không có. Staff number là opaque string được trả về trong `StaffProfileResponse` và displayed bởi STAFF-07.

---

## Files to Create/Modify

### Backend
| File | Action |
|---|---|
| `internal/model/error.go` | MODIFY — thêm ErrStaffNumberExhausted |
| `internal/repository/staff_repository.go` | MODIFY — thêm StaffNumberExists method |
| `internal/service/staff_number_service.go` | CREATE |
| `internal/service/staff_service.go` | MODIFY — inject và gọi StaffNumberService |
| `main.go` | MODIFY — wire StaffNumberService |

---

## Verification Checklist

- [ ] `StaffNumberService.Generate()` trả về string đúng 6 ký tự
- [ ] Giá trị < 100000 được zero-pad đúng (e.g., "000042", không phải "42")
- [ ] `StaffNumberExists()` trả `true` cho number đã có trong DB
- [ ] `StaffNumberExists()` trả `false` cho number chưa có
- [ ] Khi tất cả 10 attempts đều collide → `ErrStaffNumberExhausted` được trả về
- [ ] Handler trả `500 server_error` khi `ErrStaffNumberExhausted`
- [ ] `crypto/rand` được dùng (không phải `math/rand`)
- [ ] DB UNIQUE constraint trên `staff_number` là last-line defense cho race conditions
