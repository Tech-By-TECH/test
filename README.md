# create a user-mode netdev with optional SSH forward
{ 
  echo 'netdev_add user,id=net1,hostfwd=tcp::2222-:22';
  # add a virtio NIC on the hot-plug slot created earlier
  echo 'device_add virtio-net-pci,netdev=net1,id=nic1,bus=hp0';
  # verify
  echo 'info network';
  echo 'info pci';
} | nc 127.0.0.1 55555
