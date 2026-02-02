<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Restaurant Kitchen Display System</title>
    <script crossorigin src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&display=swap');
        
        body { margin: 0; font-family: 'Inter', sans-serif; -webkit-font-smoothing: antialiased; }
        ::-webkit-scrollbar { width: 10px; height: 10px; }
        ::-webkit-scrollbar-track { background: #1f2937; }
        ::-webkit-scrollbar-thumb { background: #4b5563; border-radius: 5px; }
        * { transition: all 0.2s ease; }
        .order-card:hover { transform: translateY(-2px); box-shadow: 0 10px 20px rgba(0,0,0,0.3); }
    </style>
</head>
<body>
    <div id="root"></div>

    <script type="text/babel">
        const { useState, useEffect } = React;

        // ============================================================================
        // ⚠️ CORS PROXY FIX ⚠️
        // We use this proxy to bypass the "Blocked" error from GoHighLevel
        // ============================================================================
        const GHL_API_BASE = "https://corsproxy.io/?https://services.leadconnectorhq.com";
        
        const GHL_API_KEY = "pit-9c00fde7-437e-447d-b2f0-b2dd9af7a580";
        const LOCATION_ID = "MhIt7VEPfRZ5an1Xt757";
        const PIPELINE_ID = "HqdCrlZ0NLADtJbgCyhF";

        const API_HEADERS = {
          "Authorization": `Bearer ${GHL_API_KEY}`,
          "Version": "2021-07-28",
          "Content-Type": "application/json",
          "locationId": LOCATION_ID 
        };

        const ghlApi = {
          async fetchPipeline() {
            const response = await fetch(`${GHL_API_BASE}/opportunities/pipelines/${PIPELINE_ID}`, {
              headers: API_HEADERS
            });
            if (!response.ok) throw new Error('Failed to fetch pipeline');
            return response.json();
          },

          async fetchOrders() {
            const response = await fetch(`${GHL_API_BASE}/opportunities/search?location_id=${LOCATION_ID}&pipelineId=${PIPELINE_ID}&limit=100`, {
              headers: API_HEADERS
            });
            if (!response.ok) throw new Error('Failed to fetch orders');
            return response.json();
          }
        };

        const OrderCard = ({ order }) => {
          const customerName = order.contact?.name || 'Unknown';
          const orderDetails = order.customFields?.['ORDER DETAILS'] || order.customFields?.order_details || '';
          const pickupTime = order.customFields?.['PICKUP TIME'] || order.customFields?.pickup_time || '';
          
          return (
            <div className="order-card bg-gray-800 rounded-xl p-5 mb-3 border border-gray-700 hover:border-amber-500 shadow-lg cursor-pointer">
              <div className="flex justify-between items-start mb-3">
                <div>
                  <h3 className="text-lg font-bold text-amber-400">{customerName}</h3>
                  <p className="text-xs text-gray-400 mt-1">{order.name || 'Order'}</p>
                </div>
                <span className="text-xs text-green-400 bg-green-900/30 px-2 py-1 rounded">Synced</span>
              </div>
              {orderDetails && (
                <div className="mb-3 p-3 bg-gray-900 rounded-lg border border-gray-700">
                  <p className="text-sm text-white leading-relaxed">{orderDetails}</p>
                </div>
              )}
              {pickupTime && (
                <div className="flex items-center gap-2 text-sm text-gray-300">
                  <span className="text-amber-400">🕐</span>
                  <span className="font-medium">{pickupTime}</span>
                </div>
              )}
            </div>
          );
        };

        const KanbanColumn = ({ title, orders }) => (
          <div className="bg-gray-900 rounded-xl p-4 min-h-[500px] border border-gray-800">
            <h2 className="text-lg font-bold text-amber-400 mb-4 uppercase tracking-wider flex justify-between">
              {title} <span className="text-gray-600 text-sm bg-gray-800 px-2 py-0.5 rounded-full">{orders.length}</span>
            </h2>
            <div className="space-y-3">
              {orders.length === 0 ? <p className="text-center text-gray-600 py-10">No orders</p> : orders.map(o => <OrderCard key={o.id} order={o} />)}
            </div>
          </div>
        );

        function RestaurantApp() {
          const [orders, setOrders] = useState([]);
          const [stages, setStages] = useState([]);
          const [loading, setLoading] = useState(true);
          const [error, setError] = useState(null);

          useEffect(() => {
            const loadData = async () => {
              try {
                setLoading(true);
                const pipelineData = await ghlApi.fetchPipeline();
                setStages(pipelineData.pipeline?.stages || pipelineData.stages || []);
                const ordersData = await ghlApi.fetchOrders();
                setOrders(ordersData.opportunities || []);
                setLoading(false);
              } catch (err) {
                console.error(err);
                setError(err.message);
                setLoading(false);
              }
            };
            loadData();
            const interval = setInterval(loadData, 30000); 
            return () => clearInterval(interval);
          }, []);

          const getOrdersByStage = (stageName) => {
            const stage = stages.find(s => s.name.toLowerCase().includes(stageName.toLowerCase()));
            if (!stage) return [];
            return orders.filter(o => o.pipelineStageId === stage.id);
          };

          return (
            <div className="min-h-screen bg-gray-900 text-white pb-10">
              <header className="bg-gray-800 border-b border-gray-700 px-6 py-4 sticky top-0 z-50 shadow-xl">
                <div className="flex justify-between items-center">
                  <h1 className="text-2xl font-bold text-amber-400 flex items-center gap-2"><span>🍽️</span> KDS System</h1>
                  <button onClick={() => window.location.reload()} className="text-sm bg-gray-700 px-3 py-1 rounded hover:bg-gray-600">Refresh</button>
                </div>
              </header>

              <main className="p-6">
                {error && <div className="bg-red-900/50 text-red-200 p-4 rounded mb-6 border border-red-700">Error: {error}</div>}
                
                {loading ? (
                  <div className="text-center py-20 text-gray-500 animate-pulse">Connecting to Kitchen...</div>
                ) : (
                  <div className="grid grid-cols-1 md:grid-cols-4 gap-4 overflow-x-auto">
                    <KanbanColumn title="New Order" orders={getOrdersByStage('new order')} />
                    <KanbanColumn title="Cooking" orders={getOrdersByStage('cooking')} />
                    <KanbanColumn title="Ready" orders={getOrdersByStage('ready')} />
                    <KanbanColumn title="Done" orders={getOrdersByStage('done')} />
                  </div>
                )}
              </main>
            </div>
          );
        }

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<RestaurantApp />);
    </script>
</body>
</html>
