# SQLX-WITH-GOLANG_SAMPLE
SQLX with golang and PostgreSQL sample 

### Posrgres Table Schema
```sql
create table users
(
    id       serial
        primary key,
    username varchar(50) not null
        unique
);
```

```sql
create table address
(
    id      serial
        primary key,
    user_id integer,
    city    varchar(50) not null
);
```


### FOR Nested Query
```sql
SELECT 
			users.id,
			users.username,
			COALESCE(json_agg(json_build_object('address_id', address.id, 'user_id', address.user_id, 'city', address.city)), '[]') AS addresses
		FROM users
		LEFT JOIN address ON users.id = address.user_id
		GROUP BY users.id
```

```sql
SELECT
			users.id,
			users.username,
			COALESCE(array_agg(address.id) FILTER(WHERE address.id IS NOT NULL), '{}') AS address_ids,
			COALESCE(array_agg(address.city) FILTER(WHERE address.city IS NOT NULL), '{}') AS cities
		FROM users
		LEFT JOIN address ON users.id = address.user_id
		GROUP BY users.id
```

```sql
SELECT
			users.id,
			users.username,
			COALESCE(json_agg(json_build_object('address_id', address.id, 'user_id', address.user_id, 'city', address.city)), '[]') AS addresses
		FROM users
		LEFT JOIN address ON users.id = address.user_id
		WHERE users.id = 4
		GROUP BY users.id
```


---

### Another Wat
```go
type User struct {
	ID        int       `db:"id" json:"id"`
	Username  string    `db:"username" json:"username"`
	Addresses []Address `json:"addresses"`
}

type Address struct {
	ID     int    `db:"address_id" json:"address_id"`
	UserID int    `db:"user_id" json:"user_id"`
	City   string `db:"city" json:"city"`
}

func DbFunction(cfg *config.Config) {
	db, err := sqlx.Connect("pgx", fmt.Sprintf(cfg.PGDatabaseUrl, cfg.PGDatabasePassword))
	if err != nil {
		log.Error().Err(err)
	}
	defer db.Close()

	var usersWithAddresses []User

	query := `
		SELECT 
			users.id,
			users.username,
			address.id AS address_id,
			user_id,
			city
		FROM users
		LEFT JOIN address ON users.id = address.user_id
	`

	rows, err := db.Queryx(query)
	if err != nil {
		log.Err(err)
	}
	defer rows.Close()

	userAddressMap := make(map[int]User)

	for rows.Next() {
		var user User
		var address Address

		err := rows.Scan(&user.ID, &user.Username, &address.ID, &address.UserID, &address.City)
		if err != nil {
			log.Err(err)
		}

		// Check if the user is already in the map
		if existingUser, ok := userAddressMap[user.ID]; ok {
			// Append the address to the existing user
			existingUser.Addresses = append(existingUser.Addresses, address)
			userAddressMap[user.ID] = existingUser
		} else {
			// Add a new user with the current address
			user.Addresses = append(user.Addresses, address)
			userAddressMap[user.ID] = user
		}
	}

	// Convert the map values to a slice
	for _, user := range userAddressMap {
		usersWithAddresses = append(usersWithAddresses, user)
	}

	jsonResponse, err := json.MarshalIndent(usersWithAddresses, "", "  ")
	if err != nil {
		log.Err(err)
	}

	fmt.Println(string(jsonResponse))
}
```

### Output Json Response
```json
[
   {
      "id":1,
      "username":"Porag",
      "addresses":[
         {
            "address_id":1,
            "user_id":1,
            "city":"Dhaka"
         },
         {
            "address_id":2,
            "user_id":1,
            "city":"Kishoreganj"
         }
      ]
   },
   {
      "id":2,
      "username":"Shoulin",
      "addresses":[
         {
            "address_id":3,
            "user_id":2,
            "city":"Dhaka"
         },
         {
            "address_id":4,
            "user_id":2,
            "city":"Bhagalpur"
         }
      ]
   },
   {
      "id":3,
      "username":"Faisal",
      "addresses":[
         {
            "address_id":5,
            "user_id":3,
            "city":"Kishoreganj"
         }
      ]
   },
   {
      "id":4,
      "username":"Nusrath",
      "addresses":[
         {
            "address_id":0,
            "user_id":0,
            "city":""
         }
      ]
   }
]
```




