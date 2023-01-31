# ceph single node
```
multipass launch -n ceph-node1 -v -c 4 -m 4G -d 20G --cloud-init multipass.yaml 22.04
sudo qemu-img create -f qcow2 /var/lib/libvirt/images/ceph-node1-osd0.qcow2 10G
sudo qemu-img create -f qcow2 /var/lib/libvirt/images/ceph-node1-osd1.qcow2 10G
virsh attach-disk ceph-node1 /var/lib/libvirt/images/ceph-node1-osd0.qcow2 vdc --persistent --subdriver qcow2 --live
virsh attach-disk ceph-node1 /var/lib/libvirt/images/ceph-node1-osd1.qcow2 vdd --persistent --subdriver qcow2 --live

multipass shell ceph-node1
```
```
apt update
apt install podman cephadm

cephadm bootstrap \
--cluster-network 192.168.122.0/24 \
--mon-ip $(ip route get 1 | awk '{print $7;exit}') \
--dashboard-password-noupdate \
--initial-dashboard-user admin \
--initial-dashboard-password ceph \
--allow-fqdn-hostname \
--single-host-defaults

# --skip-monitoring-stack

cat <<'EOF'> grafana.yml
service_type: grafana
spec:
  initial_admin_password: ceph
EOF

cephadm shell --mount grafana.yml:/var/lib/ceph/grafana.yml
ceph orch apply -i /var/lib/ceph/grafana.yml
ceph orch redeploy grafana
exit

apt install ceph-common

ceph -s
```
```
ceph orch apply osd --all-available-devices

# OR

ceph orch daemon add osd ceph-node1:/dev/vdc
ceph orch daemon add osd ceph-node1:/dev/vdd
```
```
ceph tell mon.* injectargs --mon_allow_pool_delete true
ceph osd pool delete libvirt-pool libvirt-pool --yes-i-really-really-mean-it

ceph osd pool create libvirt-pool
ceph osd pool application enable libvirt-pool rbd
rbd pool init libvirt-pool
rbd -p libvirt-pool create disk1 --size 1G
rbd -p libvirt-pool ls -l

radosgw-admin realm create --rgw-realm=default --default
radosgw-admin zonegroup create --rgw-zonegroup=default --master --default
radosgw-admin zone create --rgw-zonegroup=default --rgw-zone=us-east-1 --master --default

ceph orch host ls
ceph orch apply rgw default us-east-1 --placement="1 ceph-node1"
ceph -s

apt install curl unzip
curl -fsSL https://awscli.amazonaws.com/awscli-exe-linux-$(uname -m).zip -o /tmp/awscliv2.zip
unzip -q /tmp/awscliv2.zip -d /opt
/opt/aws/install -i /usr/local/aws-cli -b /usr/local/bin --update
rm /tmp/awscliv2.zip
rm -rf /opt/aws
aws --version

radosgw-admin user create --uid=test --display-name=test --access-key=test --secret-key=test
aws configure set aws_access_key_id test --profile default
aws configure set aws_secret_access_key test --profile default
aws configure set region us-east-1 --profile default

aws s3 mb s3://test --endpoint-url http://ceph-node1
aws s3 cp /etc/hosts s3://test --endpoint-url http://ceph-node1
aws s3 ls --endpoint-url http://ceph-node1
aws s3 ls s3://test --endpoint-url http://ceph-node1
```
```
apt install libvirt-daemon-system libvirt-clients libvirt-daemon-driver-storage-rbd \
  ovmf qemu-kvm qemu-utils qemu-block-extra virtinst cloud-image-utils whois --no-install-recommends
usermod -aG libvirt,kvm ubuntu

curl -fsSL https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64-disk-kvm.img \
  -o /var/lib/libvirt/images/jammy-server-cloudimg-amd64-disk-kvm.img

ceph auth del client.libvirt
ceph auth get-or-create client.libvirt mon "profile rbd" osd "profile rbd pool=libvirt-pool"
ceph auth get client.libvirt > /etc/ceph/ceph.client.libvirt.keyring

qemu-img convert -p -f qcow2 -O rbd /var/lib/libvirt/images/jammy-server-cloudimg-amd64-disk-kvm.img \
  rbd:libvirt-pool/ubuntu-template:id=libvirt

rbd snap create libvirt-pool/ubuntu-template@1
rbd snap protect libvirt-pool/ubuntu-template@1
rbd clone libvirt-pool/ubuntu-template@1 libvirt-pool/test1
qemu-img resize -f rbd "rbd:libvirt-pool/test1" 5G

ssh-keygen -C "Ubuntu Cloud User" -f ./ubuntu -q -N ""

touch my-meta-data
cat <<EOF> my-user-data
#cloud-config
password: $(printf "passw0rd" | mkpasswd -s --method=SHA-512)
chpasswd: { expire: False }
ssh_pwauth: True
ssh_authorized_keys:
  - $(cat ubuntu.pub)
EOF
cloud-localds -d raw -f iso /var/lib/libvirt/images/cidata.img my-user-data my-meta-data

qemu-img convert -p -f raw -O rbd /var/lib/libvirt/images/cidata.img rbd:libvirt-pool/cidata
```
```
SECRET_UUID=$(uuidgen)
CEPH_USER_KEY=$(ceph auth get-key client.libvirt)

cat <<EOF> secret.xml
<secret ephemeral='no' private='no'>
  <uuid>$SECRET_UUID</uuid>
  <usage type='ceph'>
    <name>client.libvirt secret</name>
  </usage>
</secret>
EOF
virsh secret-list
virsh secret-undefine
virsh secret-define --file secret.xml

virsh secret-set-value --secret $SECRET_UUID --base64 $CEPH_USER_KEY
virsh secret-get-value --secret $SECRET_UUID

cat <<EOF> pool.xml
<pool type="rbd">
  <name>ceph</name>
  <source>
    <name>libvirt-pool</name>
    <host name='ceph-node1' port='6789'/>
    <auth username='libvirt' type='ceph'>
      <secret uuid='${SECRET_UUID}'/>
    </auth>
  </source>
</pool>
EOF

virsh pool-undefine ceph
virsh pool-define pool.xml
virsh pool-start ceph
virsh pool-autostart ceph
```
```
virsh pool-refresh ceph

virt-install --connect qemu:///system \
--import \
--name test1 \
--vcpus=2 \
--ram 1024 \
--os-variant=ubuntu22.04 \
--disk vol=ceph/test1,device=disk,bus=scsi,cache=writeback,discard=unmap,format=raw,serial=$(uuidgen) \
--disk vol=ceph/cidata,device=cdrom,bus=scsi,cache=writeback,discard=unmap,format=raw,readonly=on,shareable=on \
--network network=default,model=virtio \
--controller type=scsi,model=virtio-scsi \
--virt-type kvm \
--hvm \
--accelerate \
--cpu host-passthrough \
--channel unix,target_type=virtio,name=org.qemu.guest_agent.0 \
--console pty,target_type=serial \
--graphics spice,listen=127.0.0.1 \
--serial pty \
--boot uefi \
--noautoconsole \
--noreboot
```
