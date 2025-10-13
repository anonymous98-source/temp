Great — thanks for the extra screenshots. I can see exactly what’s happening: your native query / repository is returning rows as Object[], and Spring is trying to convert them directly to your DTO/entity type which fails with:

ConverterNotFoundException: No converter found capable of converting from type [java.lang.Object[]] to type [com.tcs.usersservice.model.RoleWithPermissions]

The simplest, most reliable fix is to let the repository return the raw rows (List<Object[]>) and then map those rows to your DTOs in the service layer. That also gives you full control to group permissions per role and produce the JSON exactly like your 3rd picture.

Below I give file-by-file code you can drop into your project. I keep it simple, database-agnostic and robust (handles numbers as Number since native queries often return BigInteger or BigDecimal).


---

1) RoleDto.java (dto used in API response)

package com.tcs.usersservice.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import java.util.ArrayList;
import java.util.List;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class RoleDto {
    private Integer roleId;
    private String roleName;
    private String description;
    private String roleStatus;
    @Builder.Default
    private List<PermissionDto> permissions = new ArrayList<>();
}


---

2) PermissionDto.java

package com.tcs.usersservice.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class PermissionDto {
    private Integer id;       // MENU_ID
    private String title;     // MENU_TITLE
    private String icon;      // MENU_ICON
    private Integer order;    // MENU_ORDER
}


---

3) RoleRepository.java (Spring Data repo — return raw object[] rows)

package com.tcs.usersservice.repository;

import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.CrudRepository;
import org.springframework.stereotype.Repository;
import java.util.List;

@Repository
public interface RoleRepository extends CrudRepository<SomeRoleEntity, Integer> {
    // Native query that returns role columns + permission columns. Aliases and column order must match mapping below.
    @Query(value =
        "SELECT r.ROLE_ID, r.ROLE_NAME, r.DESCRIPTION, r.ROLE_STATUS, " +
        "p.MENU_ID, p.MENU_TITLE, p.MENU_ICON, p.MENU_ORDER " +
        "FROM ROLES r " +
        "LEFT JOIN PERMISSIONS p ON r.ROLE_ID = p.ROLE_ID " +
        "ORDER BY r.ROLE_ID, p.MENU_ORDER",
        nativeQuery = true)
    List<Object[]> findAllRolesWithPermissionsRaw();
}

Notes:

Replace SomeRoleEntity with an actual entity type in your repository generic param (or use Object and extend JpaRepository<Object, Integer> if you prefer). The native query result is read as Object[] in any case.

Column order in SELECT must match how you read Object[] in mapping code.



---

4) RoleService.java (service implementation — map Object[] → DTOs)

package com.tcs.usersservice.service;

import com.tcs.usersservice.dto.PermissionDto;
import com.tcs.usersservice.dto.RoleDto;
import com.tcs.usersservice.repository.RoleRepository;
import org.springframework.stereotype.Service;

import java.util.*;

@Service
public class RoleService {

    private final RoleRepository roleRepository;

    public RoleService(RoleRepository roleRepository){
        this.roleRepository = roleRepository;
    }

    public List<RoleDto> getAllRolesWithPermissions() {
        List<Object[]> rows = roleRepository.findAllRolesWithPermissionsRaw();

        // Use LinkedHashMap to preserve order
        Map<Integer, RoleDto> roleMap = new LinkedHashMap<>();

        for (Object[] row : rows) {
            // Expected columns in this order:
            // 0 -> ROLE_ID (Number)
            // 1 -> ROLE_NAME (String)
            // 2 -> DESCRIPTION (String)
            // 3 -> ROLE_STATUS (String)
            // 4 -> MENU_ID (Number)  -- may be null if no permission
            // 5 -> MENU_TITLE (String)
            // 6 -> MENU_ICON (String)
            // 7 -> MENU_ORDER (Number)
            Integer roleId = safeNumberToInteger(row[0]);
            String roleName = safeToString(row[1]);
            String description = safeToString(row[2]);
            String roleStatus = safeToString(row[3]);

            // find or create RoleDto
            RoleDto role = roleMap.computeIfAbsent(roleId, id -> {
                RoleDto r = RoleDto.builder()
                        .roleId(id)
                        .roleName(roleName)
                        .description(description)
                        .roleStatus(roleStatus)
                        .permissions(new ArrayList<>())
                        .build();
                return r;
            });

            // permission columns may be null (left join)
            if (row[4] != null) {
                Integer menuId = safeNumberToInteger(row[4]);
                String menuTitle = safeToString(row[5]);
                String menuIcon = safeToString(row[6]);
                Integer menuOrder = safeNumberToInteger(row[7]);

                PermissionDto p = PermissionDto.builder()
                        .id(menuId)
                        .title(menuTitle)
                        .icon(menuIcon)
                        .order(menuOrder)
                        .build();

                role.getPermissions().add(p);
            }
        }

        return new ArrayList<>(roleMap.values());
    }

    // helpers
    private static Integer safeNumberToInteger(Object o) {
        if (o == null) return null;
        if (o instanceof Number) return ((Number) o).intValue();
        try {
            return Integer.parseInt(o.toString());
        } catch (NumberFormatException e) {
            return null;
        }
    }

    private static String safeToString(Object o) {
        return o == null ? null : o.toString();
    }
}


---

5) RoleController.java (example REST endpoint returning response like your screenshot)

package com.tcs.usersservice.controller;

import com.tcs.usersservice.dto.RoleDto;
import com.tcs.usersservice.service.RoleService;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.*;

@RestController
public class RoleController {

    private final RoleService roleService;
    public RoleController(RoleService roleService) {
        this.roleService = roleService;
    }

    @GetMapping("/api/roles-with-permissions")
    public ResponseEntity<Map<String,Object>> getRolesWithPermissions() {
        List<RoleDto> roles = roleService.getAllRolesWithPermissions();

        Map<String, Object> result = new HashMap<>();
        result.put("roles", roles);

        Map<String, Object> response = new HashMap<>();
        response.put("statusCode", "OK");
        response.put("message", "OK");
        response.put("result", result);

        return ResponseEntity.ok(response);
    }
}

This returns the JSON shape exactly like your screenshot:

{
  "statusCode": "OK",
  "message": "OK",
  "result": {
    "roles": [
      {
        "roleId": 51,
        "roleName": "UCO",
        "description": "users will be responsible for managing user access",
        "roleStatus": "ACTIVE",
        "permissions": [
          { "id": 1, "title": "test1", "icon": "Dashboard", "order": 1 },
          ...
        ]
      },
      ...
    ]
  }
}


---

Important implementation notes & troubleshooting

1. Native query result types
Native queries often return numeric columns as BigInteger or BigDecimal depending on DB/driver. That's why safeNumberToInteger uses Number. Do not cast directly to Integer.


2. Column order matters
Make sure the SELECT column order in RoleRepository.findAllRolesWithPermissionsRaw() matches exactly how you read Object[] in the service.


3. If you prefer projection interfaces
Interface projections for native queries can be inconsistent across Spring Data JPA versions. The raw Object[] + manual mapping is the safest cross-DB solution and gives you control to build the exact JSON structure.


4. Remove prior mapping attempts
If you previously declared a repository method returning List<RoleWithPermissions> with nativeQuery=true, change it to List<Object[]> as shown. That is the root cause of your Converter error.


5. Null permissions
Left-join means p.* may be null for roles with no permissions — the service handles that by skipping permission creation if row[4] is null.


6. Performance
For large datasets, you may want paging or to fetch permissions separately per role. For moderate sizes this single-query + grouping approach is perfectly fine.




---

If you want, I can now:

Convert this to use Pageable and return paginated results; OR

Show a version where the repository returns List<Map<String,Object>> using JdbcTemplate (cleaner column name access); OR

Convert to named SqlResultSetMapping so JPA returns entities directly (more verbose).


Tell me which you prefer; otherwise you can paste your current repository and entity class names and I’ll adapt the exact Java package/class names for a drop-in replacement.

