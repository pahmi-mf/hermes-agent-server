# hermes-agent-server

Rencana server LLM lokal untuk Hermes Agent.

Laptop ini menjadi server `llama.cpp` (`llama-server`) di LAN. Hermes Desktop / Agent berjalan di perangkat lain: tool, file, dan shell ada di sana; laptop hanya menghasilkan token. Context dikunci di server (mulai dari 16384), bukan dibiarkan 0.

Panduan lengkapnya — spesifikasi 16 GB dan 32 GB, langkah Ubuntu Server, pengaturan Hermes, serta backup dua sisi — ada di [setup_guide.md](setup_guide.md).

Urutan dari nol di laptop — cadangan berkas, USB Ubuntu Server 26.04.1 LTS, installer, sampai Hermes di mesin lain mendapat balasan — ada di [install_ubuntu_server.md](install_ubuntu_server.md).
