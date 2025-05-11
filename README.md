# Multithread-Simulator

# Kode Program Python Multithread Simulator

`````````````````
import threading
import time
import random
import sys

class SharedMemory:
    def __init__(self):
        self.storage = {}

    def get_value(self, key):
        return self.storage.get(key, 0)

    def set_value(self, key, value):
        self.storage[key] = value

class ProcessorCache:
    def __init__(self, processor_id, main_memory):
        self.processor_id = processor_id
        self.main_memory = main_memory
        self.local_data = {}
        self.status = {}
        self.message_counter = 0
        self.total_read_duration = 0
        self.total_write_duration = 0
        self.operation_history = []

    def load_data(self, key, enable_coherence, other_caches):
        start = time.time()
        if enable_coherence and self.status.get(key) == 'Invalid':
            self.retrieve_from_main_memory(key)
        self.total_read_duration += time.time() - start
        value = self.local_data.get(key, 0)
        self.operation_history.append(f"[Processor {self.processor_id}] LOAD {key} -> {value}")
        return value

    def store_data(self, key, value, enable_coherence, other_caches):
        start = time.time()
        if enable_coherence:
            self.update_other_caches(key, other_caches)
            self.status[key] = 'Modified'
        self.local_data[key] = value
        self.main_memory.set_value(key, value)
        self.total_write_duration += time.time() - start
        self.operation_history.append(f"[Processor {self.processor_id}] STORE {key} <- {value}")

    def retrieve_from_main_memory(self, key):
        self.local_data[key] = self.main_memory.get_value(key)
        self.status[key] = 'Shared'
        self.message_counter += 1
        self.operation_history.append(f"[Processor {self.processor_id}] RETRIEVED {key} from main memory")

    def update_other_caches(self, key, other_caches):
        for cache in other_caches:
            if cache.processor_id != self.processor_id and key in cache.status and cache.status[key] != 'Invalid':
                cache.status[key] = 'Invalid'
                self.message_counter += 1
                self.operation_history.append(f"[Processor {self.processor_id}] UPDATED {key} status in Processor {cache.processor_id}")

class ProcessorThread(threading.Thread):
    def __init__(self, processor_id, use_cache_coherence, caches, operations_count):
        super().__init__()
        self.processor_id = processor_id
        self.cache = caches[processor_id]
        self.use_coherence = use_cache_coherence
        self.all_caches = caches
        self.operations_count = operations_count

    def run(self):
        for _ in range(self.operations_count):
            operation = random.choice(['read', 'write'])
            memory_location = random.choice(['x', 'y'])
            if operation == 'read':
                self.cache.load_data(memory_location, self.use_coherence, self.all_caches)
            else:
                random_value = random.randint(0, 100)
                self.cache.store_data(memory_location, random_value, self.use_coherence, self.all_caches)

def run_simulation(enable_coherence, processor_count, operations_per_processor):
    system_memory = SharedMemory()
    cache_instances = [ProcessorCache(i, system_memory) for i in range(processor_count)]
    processor_threads = [ProcessorThread(i, enable_coherence, cache_instances, operations_per_processor) for i in range(processor_count)]

    simulation_start = time.time()
    for thread in processor_threads:
        thread.start()
    for thread in processor_threads:
        thread.join()
    simulation_end = time.time()

    total_messages = sum(cache.message_counter for cache in cache_instances)
    combined_read_time = sum(cache.total_read_duration for cache in cache_instances)
    combined_write_time = sum(cache.total_write_duration for cache in cache_instances)

    return round(simulation_end - simulation_start, 6), total_messages, combined_read_time, combined_write_time, cache_instances

def compare_results(no_coherence_time, no_coherence_msgs, no_coherence_read, no_coherence_write, 
                   with_coherence_time, with_coherence_msgs, with_coherence_read, with_coherence_write, 
                   processor_count):
    time_saving = (no_coherence_time - with_coherence_time) / no_coherence_time * 100 if no_coherence_time > 0 else 0
    message_saving = (no_coherence_msgs - with_coherence_msgs) / with_coherence_msgs * 100 if with_coherence_msgs > 0 else 0

    avg_time_no = no_coherence_time / processor_count
    avg_time_with = with_coherence_time / processor_count

    avg_msgs_no = no_coherence_msgs / processor_count
    avg_msgs_with = with_coherence_msgs / processor_count

    avg_read_no = no_coherence_read / processor_count
    avg_read_with = with_coherence_read / processor_count

    avg_write_no = no_coherence_write / processor_count
    avg_write_with = with_coherence_write / processor_count

    return (time_saving, message_saving, avg_time_no, avg_time_with, 
            avg_msgs_no, avg_msgs_with, avg_read_no, avg_read_with, 
            avg_write_no, avg_write_with)

def record_operations(caches, filename='cache_operations_log.txt'):
    with open(filename, 'w') as log_file:
        for cache in caches:
            log_file.write(f"=== Processor {cache.processor_id} ===\n")
            for entry in cache.operation_history:
                log_file.write(entry + '\n')
            log_file.write("\n")

def execute_program():
    random.seed(42)
    try:
        cores = int(input("Enter number of processors (e.g. 6): "))
        operations = int(input("Enter operations per processor (e.g. 100): "))
    except ValueError:
        print("Invalid input. Please enter numbers only.")
        sys.exit(1)

    print("\n[1] Running simulation WITHOUT cache coherence protocol...")
    time_no, msgs_no, read_no, write_no, caches_no = run_simulation(False, cores, operations)

    print("[2] Running simulation WITH cache coherence protocol...")
    time_with, msgs_with, read_with, write_with, caches_with = run_simulation(True, cores, operations)

    print("\n=== PERFORMANCE COMPARISON ===")
    print(f"{'Mode':<25}{'Duration (s)':<12}{'Coherence Msgs':<18}{'Avg Time per Core (s)':<25}")
    print("-" * 70)
    print(f"{'No Coherence':<25}{time_no:<12.6f}{msgs_no:<18}{time_no/cores:.6f}")
    print(f"{'With Coherence':<25}{time_with:<12.6f}{msgs_with:<18}{time_with/cores:.6f}")
    print("-" * 70)

    results = compare_results(
        time_no, msgs_no, read_no, write_no,
        time_with, msgs_with, read_with, write_with, cores
    )

    print(f"\nTime Efficiency Improvement: {results[0]:.2f}%")
    print(f"Message Efficiency Improvement: {results[1]:.2f}%")
    print(f"Average coherence messages per core (no protocol): {results[4]:.2f}")
    print(f"Average coherence messages per core (with protocol): {results[5]:.2f}")
    print(f"Average read time per core (no protocol): {results[6]:.6f} seconds")
    print(f"Average read time per core (with protocol): {results[7]:.6f} seconds")
    print(f"Average write time per core (no protocol): {results[8]:.6f} seconds")
    print(f"Average write time per core (with protocol): {results[9]:.6f} seconds")

    record_operations(caches_with)

if __name__ == '__main__':
    execute_program()

`````````

# Output dari hasil kode programnya

![Screenshot 2025-05-11 233411](https://github.com/user-attachments/assets/9e4ba974-e6bb-431a-9041-781d3a1177c2)

``````
Enter number of processors (e.g. 6): 6
Enter operations per processor (e.g. 100): 100

[1] Running simulation WITHOUT cache coherence protocol...
[2] Running simulation WITH cache coherence protocol...

=== PERFORMANCE COMPARISON ===
Mode                     Duration (s)Coherence Msgs    Avg Time per Core (s)    
----------------------------------------------------------------------
No Coherence             0.003460    0                 0.000577
With Coherence           0.003179    10                0.000530
----------------------------------------------------------------------

Time Efficiency Improvement: 8.12%
Message Efficiency Improvement: -100.00%
Average coherence messages per core (no protocol): 0.00
Average coherence messages per core (with protocol): 1.67
Average read time per core (no protocol): 0.000020 seconds
Average read time per core (with protocol): 0.000011 seconds
Average write time per core (no protocol): 0.000020 seconds
Average write time per core (with protocol): 0.000058 seconds

``````

# Perbandingan Performa Tanpa Protokol Koherensi Cache vs Dengan Protokol Koherensi Cache

1. Simulasi Tanpa Protokol Koherensi Cache

- Pada simulasi tanpa mekanisme koherensi, sistem menunjukkan karakteristik sebagai berikut:
Waktu eksekusi total yang lebih lama (0.003460 detik) meskipun tanpa overhead protokol koherensi, menunjukkan bahwa sinkronisasi memory tetap diperlukan bahkan dalam skenario sederhana. Sistem bekerja dengan 0 pesan koherensi karena tidak ada mekanisme untuk mempertahankan konsistensi antara cache, yang bisa berbahaya untuk data yang saling bergantung. Pola waktu operasi menunjukkan keseragaman antara baca (0.000020s) dan tulis (0.000020s) per core, mengindikasikan bahwa semua operasi diperlakukan sama tanpa pertimbangan konsistensi. Ini menghasilkan perilaku yang predictable tetapi berpotensi menyebabkan race condition dan inkonsistensi data dalam lingkungan paralel nyata.

2. Simulasi Dengan Protokol Koherensi Cache

- Pada simulasi dengan mekanisme koherensi, sistem menunjukkan karakteristik sebagai berikut:
Waktu eksekusi justru lebih cepat (0.003179 detik, peningkatan 8.12%), sebuah fenomena yang kontra-intuitif tetapi dapat dijelaskan melalui optimisasi akses memory berulang dan cache locality. Sistem menghasilkan 10 pesan koherensi (1.67 pesan per core), menunjukkan overhead komunikasi yang minimal untuk mempertahankan konsistensi.
Analisis waktu operasi per core mengungkap perbedaan signifikan: waktu baca lebih cepat 45% (0.000011s) karena manfaat sharing state, sementara waktu tulis 190% lebih lambat (0.000058s) karena overhead invalidasi. Ini membuktikan trade-off klasik dalam desain sistem paralel - konsistensi yang kuat membutuhkan biaya operasi tulis yang lebih tinggi.

# Analisis Komparatif

Perbedaan utama terletak pada mekanisme penanganan konsistensi data. Tanpa koherensi, sistem lebih sederhana tetapi berisiko terhadap inkonsistensi, sementara dengan koherensi, sistem menawarkan konsistensi data dengan biaya operasi tulis yang lebih tinggi. Hasil yang tidak biasa pada waktu eksekusi total mungkin disebabkan oleh:
1. Dominasi operasi baca yang dioptimalkan protokol
2. Efek caching yang lebih efisien
3. Overhead koherensi yang terkompensasi oleh pengurangan contention
