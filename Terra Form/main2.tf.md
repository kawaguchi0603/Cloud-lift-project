provider "google" {
  # credentials に現在の認証ファイルのパスを指定
  credentials = "C:/Users/favor/AppData/Local/Packages/PythonSoftwareFoundation.Python.3.13_qbz5n2kfra8p0/LocalCache/Roaming/gcloud/application_default_credentials.json"
  
  project     = "project-a931103f-607a-470d-893"
  region      = "asia-northeast1"
  zone        = "asia-northeast1-a"
}

# 専用のVPCネットワークを作成
resource "google_compute_network" "vpc_network" {
  name                    = "lamp-gcp-vpc"
  auto_create_subnetworks = false
}

# ネットワークの中に東京エリアのサブネットを敷く
resource "google_compute_subnetwork" "subnet" {
  name          = "lamp-gcp-subnet"
  ip_cidr_range = "10.0.1.0/24"
  region        = "asia-northeast1"
  network       = google_compute_network.vpc_network.id
}

# ファイアウォール：VPC内部の全通信と、外部からのSSH（IAP等）を許可する
resource "google_compute_firewall" "allow_ssh_internal" {
  name    = "lamp-allow-ssh-internal"
  network = google_compute_network.vpc_network.name

  allow {
    protocol = "icmp"
  }
  allow {
    protocol = "tcp"
    ports    = ["22", "80", "3306"]
  }

  # GCPのIAPのIP帯とVPC内部の通信を許可
  source_ranges = ["35.235.240.0/20", "10.0.1.0/24"]
}

# ロードバランサー用 VM の作成
resource "google_compute_instance" "nginx_lb" {
  name         = "gcp-nginx-lb"
  machine_type = "e2-micro"

  boot_disk {
    initialize_params {
      image = "ubuntu-os-cloud/ubuntu-2204-lts"
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.subnet.id
    access_config {}
  }

  # 全自動インストールを定義
  metadata_startup_script = "curl -fsSL https://tailscale.com/install.sh | sh"
}

# MySQL Master 用 VM の作成
resource "google_compute_instance" "mysql_master" {
  name         = "gcp-mysql-master"
  machine_type = "e2-medium"

  boot_disk {
    initialize_params {
      image = "ubuntu-os-cloud/ubuntu-2204-lts"
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.subnet.id
    access_config {}
  }

  metadata_startup_script = "curl -fsSL https://tailscale.com/install.sh | sh"
}

# MySQL Slave 用 VM の作成
resource "google_compute_instance" "mysql_slave" {
  name         = "gcp-mysql-slave"
  machine_type = "e2-medium"

  boot_disk {
    initialize_params {
      image = "ubuntu-os-cloud/ubuntu-2204-lts"
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.subnet.id
    access_config {}
  }

  metadata_startup_script = "curl -fsSL https://tailscale.com/install.sh | sh"
}
