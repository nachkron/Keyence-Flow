import socket
import time
import csv
import pandas as pd
import streamlit as st
import plotly.graph_objects as go
from plotly.subplots import make_subplots
from datetime import datetime
import os

# Configuration
LINE_NUM = 1
KEYENCE_IP = "196.168.0.100"  # Change this to your Keyence device IP address
KEYENCE_PORT = 502
TIMEOUT = 5
MODBUS_COMMAND = bytes.fromhex("000100000006010400020010")

# Initialize session state
if 'data' not in st.session_state:
    st.session_state.data = pd.DataFrame(columns=[
        'Timestamp', 'Instantaneous Flow', 'Accumulated Flow', 
        'Temperature 1', 'Temperature 2'
    ])
if 'running' not in st.session_state:
    st.session_state.running = False
if 'csv_filename' not in st.session_state:
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    st.session_state.csv_filename = f"Flow_rate_Line_{LINE_NUM}_{timestamp}.csv"

def read_modbus_data(ip, port, command):
    """Read data from Keyence device via Modbus TCP"""
    try:
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.settimeout(TIMEOUT)
        sock.connect((ip, port))
        sock.sendall(command)
        response = sock.recv(1024)
        sock.close()
        
        if len(response) >= 21:  # Ensure we have enough data
            register_data = response[9:]
            
            # Parse values
            inst_flow_raw = int.from_bytes(register_data[0:4], 'big')
            accum_flow = int.from_bytes(register_data[4:8], 'big')
            temp1_raw = int.from_bytes(register_data[8:10], 'big')
            temp2_raw = int.from_bytes(register_data[10:12], 'big')
            
            # Apply scaling factors
            inst_flow = inst_flow_raw * 0.01
            temp1 = temp1_raw * 0.1
            temp2 = temp2_raw * 0.1
            
            return {
                'Instantaneous Flow': inst_flow,
                'Accumulated Flow': accum_flow,
                'Temperature 1': temp1,
                'Temperature 2': temp2
            }
        return None
    except Exception as e:
        st.error(f"Error reading data: {e}")
        return None

def append_to_csv(data_dict, filename):
    """Append data to CSV file"""
    file_exists = os.path.isfile(filename)
    
    with open(filename, 'a', newline='') as f:
        writer = csv.DictWriter(f, fieldnames=['Timestamp', 'Instantaneous Flow', 
                                                'Accumulated Flow', 'Temperature 1', 
                                                'Temperature 2'])
        if not file_exists:
            writer.writeheader()
        
        row = {'Timestamp': datetime.now().strftime("%Y-%m-%d %H:%M:%S")}
        row.update(data_dict)
        writer.writerow(row)

def create_graphs(df):
    """Create real-time graphs"""
    if df.empty:
        return None
    
    fig = make_subplots(
        rows=3, cols=1,
        subplot_titles=('Instantaneous Flow', 'Accumulated Flow', 'Temperatures'),
        vertical_spacing=0.1,
        specs=[[{"secondary_y": False}],
               [{"secondary_y": False}],
               [{"secondary_y": False}]]
    )
    
    # Instantaneous Flow
    fig.add_trace(
        go.Scatter(x=df['Timestamp'], y=df['Instantaneous Flow'],
                   name='Instantaneous Flow', line=dict(color='blue', width=2)),
        row=1, col=1
    )
    
    # Accumulated Flow
    fig.add_trace(
        go.Scatter(x=df['Timestamp'], y=df['Accumulated Flow'],
                   name='Accumulated Flow', line=dict(color='green', width=2)),
        row=2, col=1
    )
    
    # Temperatures
    fig.add_trace(
        go.Scatter(x=df['Timestamp'], y=df['Temperature 1'],
                   name='Temperature 1', line=dict(color='red', width=2)),
        row=3, col=1
    )
    fig.add_trace(
        go.Scatter(x=df['Timestamp'], y=df['Temperature 2'],
                   name='Temperature 2', line=dict(color='orange', width=2)),
        row=3, col=1
    )
    
    fig.update_xaxes(title_text="Time", row=3, col=1)
    fig.update_yaxes(title_text="Flow", row=1, col=1)
    fig.update_yaxes(title_text="Flow", row=2, col=1)
    fig.update_yaxes(title_text="°C", row=3, col=1)
    
    fig.update_layout(height=900, showlegend=True)
    
    return fig

# Streamlit UI
st.set_page_config(page_title=f"Flow Meter Monitor - Line {LINE_NUM}", layout="wide")
st.title(f"🌊 Keyence Flow Meter Monitor - Line {LINE_NUM}")

# Sidebar - Configuration
with st.sidebar:
    st.header("⚙️ Configuration")
    keyence_ip = st.text_input("Keyence IP Address", value=KEYENCE_IP)
    polling_interval = st.slider("Polling Interval (seconds)", 1, 60, 5)
    
    st.divider()
    st.header("📊 Data File")
    st.text(st.session_state.csv_filename)
    
    st.divider()
    st.header("🎛️ Controls")
    
    col1, col2 = st.columns(2)
    with col1:
        if st.button("▶️ Start", use_container_width=True):
            st.session_state.running = True
    with col2:
        if st.button("⏸️ Stop", use_container_width=True):
            st.session_state.running = False
    
    if st.button("🗑️ Clear Data", use_container_width=True):
        st.session_state.data = pd.DataFrame(columns=[
            'Timestamp', 'Instantaneous Flow', 'Accumulated Flow', 
            'Temperature 1', 'Temperature 2'
        ])
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        st.session_state.csv_filename = f"Flow_rate_Line_{LINE_NUM}_{timestamp}.csv"
        st.rerun()

# Main content area
placeholder = st.empty()

# Auto-refresh when running
if st.session_state.running:
    data = read_modbus_data(keyence_ip, KEYENCE_PORT, MODBUS_COMMAND)
    
    if data:
        # Add timestamp
        new_row = pd.DataFrame([{
            'Timestamp': datetime.now(),
            **data
        }])
        
        st.session_state.data = pd.concat([st.session_state.data, new_row], ignore_index=True)
        
        # Append to CSV
        append_to_csv(data, st.session_state.csv_filename)
        
        st.success("✅ Data updated successfully!")
    else:
        st.error("❌ Failed to read data from device")
    
    time.sleep(polling_interval)
    st.rerun()

# Display current values
if not st.session_state.data.empty:
    latest = st.session_state.data.iloc[-1]
    
    col1, col2, col3, col4 = st.columns(4)
    
    with col1:
        st.metric("Instantaneous Flow", f"{latest['Instantaneous Flow']:.2f}")
    with col2:
        st.metric("Accumulated Flow", f"{latest['Accumulated Flow']:.0f}")
    with col3:
        st.metric("Temperature 1", f"{latest['Temperature 1']:.1f} °C")
    with col4:
        st.metric("Temperature 2", f"{latest['Temperature 2']:.1f} °C")
    
    st.divider()
    
    # Display graphs
    st.subheader("📈 Real-time Graphs")
    fig = create_graphs(st.session_state.data)
    if fig:
        st.plotly_chart(fig, use_container_width=True)
    
    st.divider()
    
    # Display data table
    st.subheader("📋 Data Table")
    st.dataframe(st.session_state.data, use_container_width=True)
    
    # Export buttons
    st.divider()
    col1, col2 = st.columns(2)
    
    with col1:
        csv_data = st.session_state.data.to_csv(index=False).encode('utf-8')
        st.download_button(
            label="📥 Download CSV",
            data=csv_data,
            file_name=st.session_state.csv_filename,
            mime='text/csv',
            use_container_width=True
        )
    
    with col2:
        if fig:
            img_bytes = fig.to_image(format="png", width=1920, height=1080)
            st.download_button(
                label="📥 Download Graph (PNG)",
                data=img_bytes,
                file_name=f"Flow_rate_Line_{LINE_NUM}_graph_{datetime.now().strftime('%Y%m%d_%H%M%S')}.png",
                mime='image/png',
                use_container_width=True
            )
else:
    st.info("👆 Click 'Start' to begin monitoring")
