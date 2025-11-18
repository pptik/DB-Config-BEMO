Ini adalah skenario umum. Jika sudah memiliki data `rute`, `halte`, dan `halte_urutan`, (generate) data untuk `halte_vertices` dan `rute_edges` secara otomatis menggunakan SQL.

## Langkah 1: Mengisi Tabel `halte_vertices`

Kueri ini akan mengambil setiap `halte_id` (misal: 'H001') dari tabel `halte` Anda dan memasukkannya ke `halte_vertices`. Kolom `id` (integer) akan terisi otomatis berkat `BIGSERIAL`.

```sql
-- Mengisi tabel "perekat" (glue table)
-- Ini memetakan ID halte (varchar) ke ID simpul (integer)
INSERT INTO public.halte_vertices (halte_id)
SELECT id
FROM public.halte
ORDER BY id;
```

## Langkah 2: Mengisi `rute_edges` (Tipe 'NAIK')

```sql
WITH
lagged_routes AS (
  SELECT
    route_id,
    halte_id AS source_halte_id,
    LEAD(halte_id) OVER (PARTITION BY route_id ORDER BY urutan ASC) AS target_halte_id
  FROM
    public.halte_urutan
),

edges_with_data AS (
  SELECT
    lr.route_id,

    -- Dapatkan ID Integer (Integer ID)
    hv_source.id AS source_vertex_id,
    hv_target.id AS target_vertex_id,

    -- Dapatkan Lokasi (Geospatial)
    h_source.location AS source_location,
    h_target.location AS target_location

  FROM
    lagged_routes lr
  -- Join untuk mendapatkan ID Integer Source
  JOIN public.halte_vertices hv_source ON lr.source_halte_id = hv_source.halte_id
  -- Join untuk mendapatkan ID Integer Target
  JOIN public.halte_vertices hv_target ON lr.target_halte_id = hv_target.halte_id
  -- Join untuk mendapatkan Lokasi Source
  JOIN public.halte h_source ON lr.source_halte_id = h_source.id
  -- Join untuk mendapatkan Lokasi Target
  JOIN public.halte h_target ON lr.target_halte_id = h_target.id

  WHERE
    -- Hapus baris terakhir dari setiap rute (dimana target_halte_id IS NULL)
    lr.target_halte_id IS NOT NULL
),

calculated_edges AS (
  SELECT
    route_id,
    source_vertex_id,
    target_vertex_id,

    ST_Distance(source_location, target_location) AS distance_m,
    (ST_Distance(source_location, target_location) / 333.0) AS cost_minutes

  FROM
    edges_with_data
)

INSERT INTO public.rute_edges (source, target, cost, reverse_cost, route_id, tipe, distance_meters)
SELECT
  source_vertex_id,
  target_vertex_id,
  cost_minutes,
  -1,
  route_id,
  'NAIK'::edge_tipe,
  distance_m
FROM
  calculated_edges;
```

## Langkah 3: Mengisi `rute_edges` (Tipe 'JALAN' - Transit)

```sql
WITH
all_halte_vertices AS (
  SELECT
    h.id AS halte_id,
    h.location,
    hv.id AS vertex_id
  FROM
    public.halte h
  JOIN public.halte_vertices hv ON h.id = hv.halte_id
),

nearby_pairs AS (
  SELECT
    a.vertex_id AS source_vertex_id,
    b.vertex_id AS target_vertex_id,
    ST_Distance(a.location, b.location) AS distance_m
  FROM
    all_halte_vertices a
  JOIN
    all_halte_vertices b ON ST_DWithin(a.location, b.location, 500)
  WHERE
    a.vertex_id != b.vertex_id
)

INSERT INTO public.rute_edges (source, target, cost, reverse_cost, route_id, tipe, distance_meters)
SELECT
  source_vertex_id,
  target_vertex_id,

  (distance_m / 83.0) * 50.0 AS walk_cost,
  (distance_m / 83.0) * 50.0 AS reverse_walk_cost,

  NULL,
  'JALAN'::edge_tipe,
  distance_m
FROM
  nearby_pairs
WHERE
  distance_m > 0;
```

Setelah menjalankan 3 langkah ini, tabel `halte_vertices` dan `rute_edges` akan terisi penuh dan siap digunakan oleh backend.
