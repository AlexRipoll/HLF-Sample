## cryptogen

Generate crypto-config.yaml template:

    `cryptogen showtemplate > crypto-config.yaml`

Generate crypto material:

    `cryptogen generate --config=<crypto-config.yaml file path> --output="<output folder name>"`

Add components to existing setup (without affecting the current crypto material):

    `cryptogen extend --config=<extension config file>` --input=<input folder>

## configtxgen

Generate genesis block (FABRIC_CFG_PATH env variable needs to be specified)

    `configtxgen -profile <profile name set in configtx.yaml> -channelID <channel name> -outputBlock <full path file name>`

Inspect content of a block (from binary to JSON)

    `configtxgen -inspectBlock <blockfile name> > <file name>.json`

Generate Application Channel Tx

    `configtxgen -outputCreateChannelTx <file name> > -profile <profile name set in configtx.yaml> -channelID <channel name>

Inspect content of a Tx in JSON

    `configtxgen -inspectChannelCreateTx <tx file name> > <file name>.json`



