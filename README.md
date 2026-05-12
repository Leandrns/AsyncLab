# ⚡ AsyncLab

## 🧪 Laboratório Async

### 🎯 Objetivo
Analisar o programa e tornar a sua execução **assíncrona**.

---

## 👥 Membros do Grupo
Caio Alexandre dos Santos - RM: 558460

Leandro do Nascimento Souza - RM: 558893

Rafael de Mônaco Maniezo - RM: 556079

Vinicius Rozas Panucci de Paula Cont - RM: 555338

---

## 🛠️ Modificações Realizadas

Para tornar o programa assíncrono, foram identificados e refatorados os seguintes pontos:

### 1. Download do CSV

O método `WebClient.DownloadFile` foi substituído por `HttpClient` com `await GetByteArrayAsync`, liberando a thread principal durante a espera da resposta de rede em vez de mantê-la bloqueada.

```csharp
// Antes
using (var wc = new WebClient())
    wc.DownloadFile(CSV_URL, tempCsvPath);

// Depois
using var http = new HttpClient();
var bytes = await http.GetByteArrayAsync(CSV_URL);
await File.WriteAllBytesAsync(tempCsvPath, bytes);
```

---

### 2. Leitura do arquivo CSV

`File.ReadAllLines` foi substituído por `File.ReadAllLinesAsync`, tornando a leitura do disco não bloqueante.

```csharp
// Antes
var linhas = File.ReadAllLines(tempCsvPath, Encoding.UTF8);

// Depois
var linhas = await File.ReadAllLinesAsync(tempCsvPath, Encoding.UTF8);
```

---

### 3. Escrita dos arquivos CSV e JSON por UF

As escritas síncronas com `StreamWriter.WriteLine` e `File.WriteAllText` foram substituídas pelas versões assíncronas `WriteLineAsync` e `WriteAllTextAsync`, evitando que operações de disco bloqueiem a thread.

```csharp
// Antes
swOut.WriteLine($"{m.Tom};{m.Ibge};...");
File.WriteAllText(jsonPath, json, Encoding.UTF8);

// Depois
await swOut.WriteLineAsync($"{m.Tom};{m.Ibge};...");
await File.WriteAllTextAsync(jsonPath, json, Encoding.UTF8);
```

---

### 4. Processamento das UFs em paralelo

O `foreach` sequencial que processava uma UF por vez foi substituído por `Parallel.ForEachAsync`, permitindo que múltiplas UFs sejam processadas simultaneamente. Como cada UF é completamente independente das demais, essa paralelização é segura e representa o **maior ganho de desempenho** observado.

```csharp
// Antes
foreach (var uf in ufsOrdenadas)
{
    // processa uma UF por vez...
}

// Depois
await Parallel.ForEachAsync(
    ufsOrdenadas,
    new ParallelOptions { MaxDegreeOfParallelism = Environment.ProcessorCount },
    async (uf, ct) =>
    {
        // processa múltiplas UFs ao mesmo tempo...
    });
```

---

### 5. Cálculo de hashes PBKDF2 em paralelo por UF

Dentro de cada UF, o cálculo de hash dos municípios — operação CPU-bound com 50.000 iterações — era feito sequencialmente. Foi refatorado com `Task.Run` + `AsParallel().AsOrdered()`, distribuindo o trabalho entre os núcleos disponíveis do processador. A escrita nos arquivos permanece sequencial após os cálculos, pois `StreamWriter` não é thread-safe.

```csharp
// Antes
foreach (var m in listaUf)
{
    string hashHex = Util.DeriveHashHex(password, salt, PBKDF2_ITERATIONS, HASH_BYTES);
    swOut.WriteLine(...);
}

// Depois
var resultados = await Task.Run(() =>
    listaUf
        .AsParallel()
        .AsOrdered()
        .Select(m =>
        {
            string password = m.ToConcatenatedString();
            byte[] salt = Util.BuildSalt(m.Ibge);
            string hashHex = Util.DeriveHashHex(password, salt, PBKDF2_ITERATIONS, HASH_BYTES);
            return (m, hashHex);
        })
        .ToList()
);

foreach (var (m, hashHex) in resultados)
{
    await swOut.WriteLineAsync(...);
}
```

---

## 📊 Impactos Observados no Tempo de Execução

| Modificação | Tipo | Impacto |
|---|---|---|
| Download com `HttpClient` async | I/O-bound | Thread principal liberada durante a espera de rede |
| `ReadAllLinesAsync` | I/O-bound | Leitura de disco não bloqueante |
| `WriteLineAsync` / `WriteAllTextAsync` | I/O-bound | Escrita de disco não bloqueante |
| UFs em paralelo (`Parallel.ForEachAsync`) | CPU + I/O | **Maior ganho** — todas as UFs processadas simultaneamente |
| PBKDF2 em paralelo por UF (`AsParallel`) | CPU-bound | Uso de todos os núcleos do processador por UF |

O ganho mais expressivo foi observado no **processamento paralelo das UFs**, já que o programa original utilizava apenas um núcleo da CPU para um trabalho naturalmente paralelizável.

---

👨‍🏫 © 2025 | Professor Vinícius Costa Santos
